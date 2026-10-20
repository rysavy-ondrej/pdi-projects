# Project 4: Fault-Tolerant Multi-Agent Alert Triage on Dapr

| | |
| --- | --- |
| **Main technologies** | Dapr (Workflow, Pub/Sub, State Store; optionally Dapr Agents), Redis, Docker Compose or Kubernetes |
| **AI/ML component** | LLM-based agents (an LLM API or a local LLM via Ollama) |
| **Data** | 20–30 small incident bundles prepared by the students from public datasets |
| **Derived from** | Uncertainty-Aware Multi-Agent System for Network Alert Triage (GJU) |

## Motivation

Security teams receive many alerts, and each must be triaged: is it benign, suspicious, or malicious? Multi-agent LLM systems split this work among specialised agents, e.g., one agent examines network evidence, another checks threat intelligence, and a third reviews the conclusion.

Such systems are **distributed systems**: agents run as separate services, exchange messages, keep state, and call slow, costly, and sometimes failing external models. Crashes and retries can cause lost work, duplicate (paid) LLM calls, or inconsistent conclusions. Dapr provides building blocks (workflows, pub/sub, state) for building such systems reliably.

This project focuses on the **architecture and reliability** of a multi-agent triage system, not on the accuracy of its decisions.

## Key Concepts

- **Dapr sidecar:** each service runs next to a Dapr process that provides building blocks over HTTP/gRPC: messaging, state, workflows, secrets. The service code does not depend on the concrete infrastructure (e.g., Redis or Kafka).
- **Orchestration:** a central coordinator (here a Dapr Workflow) calls the agents in a defined order and collects their results.
- **Choreography:** there is no coordinator; agents react to events published on topics and publish their own results.
- **Durable workflow:** Dapr Workflow saves the progress of a workflow. After a crash it *replays* the workflow code and skips steps that have already completed. Activities (e.g., an LLM call) may still run more than once, so they must be idempotent.
- **Idempotency key:** a unique identifier of an operation (e.g., `incident-17/evidence-agent/attempt-1`). If the operation is repeated with the same key, the stored result is returned instead of executing it again.
- **Resiliency policies:** Dapr configuration of retries, timeouts, and circuit breakers for calls between services and to components.

## Research Question

What are the latency, cost, and reliability trade-offs between orchestrated and choreographed multi-agent architectures, and how do they behave under failures?

## System Overview

```text
Orchestration                              Choreography

 incident ──► Workflow (Dapr)              incident ──► topic "incidents"
               │  fan-out                                │
      ┌────────┼────────┐                   ┌────────────┴───────────┐
      ▼        ▼        ▼                   ▼                        ▼
  evidence  threat-   (more agents)     evidence agent        threat-intel agent
   agent    intel                           │ topic "findings"       │
      └────────┼────────┘                   └───────────┬────────────┘
               ▼ fan-in                                 ▼
           critic agent                            critic agent
               ▼                                (waits for all findings
         final decision                          in the state store)
                                                        ▼
                                                  final decision

 Both: every agent input/output is appended to an audit log.
```

## Data

- An **incident bundle** is a small JSON file with an alert (e.g., "possible C2 beaconing from 10.0.0.5") and the related telemetry: a few dozen Zeek or flow log lines and, if relevant, domain or IP reputation data.
- Prepare 20–30 bundles for 2 scenarios (e.g., C2 beaconing and port scanning), including benign look-alikes, from the sources below.
- Each bundle has an expected label (benign / suspicious / malicious) so that the outcome can be checked, but accuracy is reported only to show that the system works.

| Dataset | Use | Download |
| --- | --- | --- |
| CTU-13 | Malicious telemetry (botnet C2, scanning, spam); labelled flows and botnet PCAPs | [Dataset page](https://www.stratosphereips.org/datasets-ctu13); full archive (1.9 GB) at `https://mcfp.felk.cvut.cz/publicDatasets/CTU-13-Dataset/CTU-13-Dataset.tar.bz2` |
| Stratosphere normal captures | Benign telemetry for benign look-alike incidents | [Dataset page](https://www.stratosphereips.org/datasets-normal) |
| abuse.ch Feodo Tracker | Botnet C2 IP blocklist for the threat-intelligence agent | [feodotracker.abuse.ch/blocklist](https://feodotracker.abuse.ch/blocklist/) |
| abuse.ch URLhaus | Malicious URLs and domains for the threat-intelligence agent | [urlhaus.abuse.ch](https://urlhaus.abuse.ch) |

- Download the reputation lists once and store a snapshot with the bundles, so that runs are reproducible and do not depend on live services. Note that current blocklists will rarely contain the (old) CTU-13 indicators; you may add the known C2 addresses of the scenarios to the local list and document this.
- Synthetic telemetry from a documented generator is also acceptable.

## Tasks

### Required Tasks

1. **Agents** as separate Dapr services (Python):
   - **evidence agent:** summarises the telemetry and lists indicators,
   - **threat-intelligence agent:** checks indicators against a local reputation list or a small offline dataset (no live external APIs required),
   - **critic agent:** reviews the findings and produces the final decision: benign, suspicious, malicious, or insufficient evidence.
2. **Single-agent baseline:** one LLM call with the whole bundle.
3. **Two architectures:**
   - **Orchestration:** Dapr Workflow with fan-out to the evidence and threat-intelligence agents, fan-in, then the critic.
   - **Choreography:** agents communicate only via Dapr pub/sub topics; partial results are collected in a Dapr state store.
4. **Idempotent LLM calls:** each LLM call has an idempotency key; results are stored in the state store and reused on repetition.
5. **Audit trail:** an append-only log (e.g., a Kafka or Redis stream topic) recording every agent input, output, and decision with the incident ID and timestamps.
6. **Mock LLM:** a small service with the same API as the real LLM that returns recorded responses with configurable latency and error rate. Use it for failure and load experiments so that they are cheap and reproducible; use the real LLM for the baseline and the demo.

### Optional Tasks

- The agents implemented with the Dapr Agents framework.
- A larger agent set (e.g., a separate triage agent that decides which specialists to call).

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Architecture | Single agent vs orchestration vs choreography | End-to-end latency per incident, number of LLM calls and tokens | What does the multi-agent design cost, and which architecture is faster? |
| Failures | Kill an agent or its sidecar during an investigation | Completed vs lost incidents, recovery time, consistency of the audit trail | Which architecture recovers better from crashes? |
| Idempotency | With vs without idempotency keys, under injected failures | Number of duplicate LLM calls, duplicate decisions | Do retries and workflow replays cause repeated work? |
| Load | Number of concurrent incidents (mock LLM) | Throughput, queueing delay, p99 latency | Where is the bottleneck under load? |

### Optional Experiments

| Experiment | What to vary | What to measure |
| --- | --- | --- |
| Resiliency policies | Retry, timeout, circuit-breaker settings (mock LLM with errors) | Success rate, latency tail |

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: all agent services with their Dapr sidecars, the state store, and the pub/sub broker.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- Agent services, the orchestration (Workflow) and choreography (pub/sub) implementations, and the single-agent baseline.
- Dapr component and resiliency configuration files.
- Mock LLM service with the recorded responses.
- Incident bundles with expected labels and snapshots of the reputation lists.
- Failure-injection and load scripts.
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Architecture:** table with, for the single agent, orchestration, and choreography, the mean and p95 latency per incident, the number of LLM calls and tokens per incident, the estimated cost, and the share of correct decisions.
2. **Failures:** table with, for each failure scenario and architecture, completed and lost incidents, recovery time, and completeness of the audit trail.
3. **Idempotency:** table of duplicate LLM calls and duplicate decisions with and without idempotency keys under injected failures.
4. **Load:** plot of throughput and p99 latency versus the number of concurrent incidents.
5. **Audit trail:** an excerpt of the complete audit trail of one incident.

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of Dapr and the building blocks used in the project: sidecars, Workflow durability and replay, pub/sub delivery guarantees, state store concurrency, resiliency policies.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- one incident processed by both architectures, with the final decision,
- killing an agent during an investigation and showing that the incident still completes,
- the audit trail of the incident.

## Practical Tips

- Start with orchestration; it is easier to debug. Add choreography once the agents work.
- In choreography, decide early how the critic knows that all findings have arrived (e.g., a counter in the state store with optimistic concurrency).
- Use a local LLM (Ollama) or the mock LLM during development to avoid costs and rate limits.
- Kill containers with `docker kill` (not graceful stop) to simulate crashes.

## References

- [Dapr documentation](https://docs.dapr.io): *Workflow*, *Pub/Sub*, *State management*, *Resiliency*.
- [Dapr Agents](https://github.com/dapr/dapr-agents).
- [Ollama](https://ollama.com).
