# Project 6: Stream Ingestion of LLM-Extracted Facts into a Knowledge Graph

| | |
| --- | --- |
| **Main technologies** | Apache Kafka or Dapr Pub/Sub, Neo4j, Docker Compose or Kubernetes |
| **AI/ML component** | LLM-based information extraction (an LLM API or a local LLM via Ollama) |
| **Data** | 100–200 public threat reports or security advisories |
| **Derived from** | LLM-Assisted Cyber Threat Intelligence Knowledge Graphs (PSUT) |

## Motivation

Threat reports describe which actors use which malware and exploit which vulnerabilities, but only as free text. LLMs can extract these statements as structured facts and store them in a knowledge graph. LLM extraction, however, is slow, rate-limited, non-deterministic, and sometimes wrong.

Feeding such facts into a shared knowledge graph raises classic distributed-systems problems: duplicate delivery, concurrent writers, ordering, backpressure, and reprocessing. This project builds a reliable ingestion pipeline and studies these effects.

## Role of the LLM

The LLM performs **information extraction**: it turns unstructured report text into structured facts (entity–relation–entity triples). For example, from the sentence

> "APT28 used the X-Agent implant to exfiltrate data via CVE-2023-23397 in Microsoft Outlook."

it produces JSON such as:

```json
[
  {"subject": {"type": "ThreatActor", "name": "APT28"},
   "relation": "USES",
   "object": {"type": "Malware", "name": "X-Agent"},
   "evidence": "APT28 used the X-Agent implant",
   "confidence": 0.9},
  {"subject": {"type": "ThreatActor", "name": "APT28"},
   "relation": "EXPLOITS",
   "object": {"type": "Vulnerability", "name": "CVE-2023-23397"},
   "evidence": "via CVE-2023-23397 in Microsoft Outlook",
   "confidence": 0.85}
]
```

which becomes `(APT28)-[:USES]->(X-Agent)` and `(APT28)-[:EXPLOITS]->(CVE-2023-23397)` in the graph.

The LLM is used as-is with a fixed prompt. For this project it matters mainly as an **unreliable external service** whose properties cause the problems studied:

| LLM property | Resulting problem | Experiment |
| --- | --- | --- |
| Slow (seconds per call) and rate-limited | Queue lag, backpressure, worker scaling | Backpressure |
| Fails or times out (HTTP 429, 5xx) | Retries, which cause duplicate delivery | Duplicates |
| Non-deterministic | Re-extraction yields different facts, so naive idempotency breaks | Duplicates, Reprocessing |
| Inconsistent naming ("APT28" / "Fancy Bear" / "Sofacy") | Alias resolution, hot entities, ordering effects | Concurrency (optional) |
| Hallucinations, schema violations | Validation, dead-letter queue | Validation |

## Key Concepts

- **At-least-once delivery:** a message is delivered one or more times; consumers must tolerate duplicates.
- **Idempotent write:** writing the same fact twice leaves the graph unchanged. In Neo4j this is done with `MERGE` on a unique key.
- **Backpressure:** slowing down producers or buffering when consumers (here the LLM workers) cannot keep up; visible as growing *consumer lag* in Kafka.
- **Dead-letter queue (DLQ):** a topic where messages that cannot be processed are stored for inspection instead of being retried forever.
- **Provenance:** for each fact, where it came from (report, chunk, model, prompt version) and with what confidence.

## Research Question

How can an ingestion pipeline turn slow, rate-limited, and non-deterministic LLM output into a consistent, deduplicated knowledge graph?

## System Overview

```text
Reports ──► Chunker ──► topic "chunks" ──► Extraction workers (×N) ──► topic "candidate-facts"
                                                 │      ▲                       │
                                                 ▼      │                       ▼
                                          LLM (real or mock)               Validator
                                          + stored outputs              + alias normaliser
                                                                         │            │
                                                                     valid facts   rejected
                                                                         ▼            ▼
                                                               Neo4j writer (MERGE)  DLQ topic
```

## Data

| Dataset | Use | Download |
| --- | --- | --- |
| CISA cybersecurity advisories | Main corpus of threat reports (HTML pages, many with PDF versions) | [cisa.gov/news-events/cybersecurity-advisories](https://www.cisa.gov/news-events/cybersecurity-advisories) |
| MITRE ATT&CK (Enterprise, STIX 2.1) | Alias table (group and software names with their `aliases` / `x_mitre_aliases` fields); allowed entity names | [github.com/mitre-attack/attack-stix-data](https://github.com/mitre-attack/attack-stix-data) (file `enterprise-attack/enterprise-attack.json`) |
| ATT&CK references (optional) | Further public reports: the `external_references` of ATT&CK groups and software point to vendor reports | same STIX file |

- Collect 100–200 reports with a small download script and store the raw text, the source URL, and the download date; the collected corpus is part of the deliverables, so that experiments can be repeated even if the web pages change.
- Use only publicly available reports and respect the sites' terms of use.

## Tasks

### Required Tasks

1. **Pipeline:**
   1. reports are split into chunks that fit the LLM context; each chunk is one message in the queue,
   2. parallel extraction workers call the LLM and produce candidate facts (JSON),
   3. a validator checks facts against schema rules: allowed entity and relation types, and an evidence quote that must appear verbatim in the source chunk (a cheap, deterministic hallucination check),
   4. entity names are normalised using the alias table,
   5. valid facts are written with idempotent upserts (`MERGE`) into Neo4j,
   6. rejected facts go to a dead-letter queue with the rejection reason.
2. **Provenance:** store the source report, chunk, model, prompt version, and confidence with each fact.
3. **Idempotency under non-determinism:** `MERGE` is idempotent only if a retry produces the same fact. Persist each LLM output keyed by `(report hash, chunk id, model, prompt version)`; a retry after a failed write reuses the stored output instead of calling the LLM again. Fact identity in the graph is based on normalised content (`subject | relation | object`), not on the extraction run.
4. **Mock LLM:** extract the corpus once with the real LLM and record the responses. For the system experiments, use a mock LLM service that replays the recorded responses with configurable latency, rate limit, error rate, and output variation. This keeps experiments cheap and reproducible.
5. **Model choice:** an LLM API (real rate limits and cost) or a local LLM via Ollama (throughput limited by your hardware). State your choice; it determines what "rate limit" means in the experiments.

### Optional Tasks

- Implement the pipeline in both Kafka and Dapr Pub/Sub and compare them.
- Multiple concurrent Neo4j writers.

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Backpressure | Number of workers, LLM rate limit and latency (mock LLM) | Throughput, consumer lag, LLM error rate | How many workers are useful for a given rate limit? |
| Duplicates | At-least-once delivery with injected failures; with/without idempotency keys and stored LLM outputs | Duplicate nodes and edges | Which mechanisms are needed for a duplicate-free graph? |
| Reprocessing | Replay from an offset after a change in the validator rules | Time to rebuild, consistency of the graph | Can the graph be rebuilt reliably when rules change? |
| Validation and quality | Pipeline stage: raw LLM output, after validation, after normalisation and deduplication | Share of facts rejected and reasons; precision of facts on a small hand-labelled sample (10–20 reports) | How much do validation and normalisation improve the graph? |

### Optional Experiments

| Experiment | What to vary | What to measure |
| --- | --- | --- |
| Concurrency | Number of writers, hot entities (popular malware families) | Lock contention, write latency, failed transactions |
| Ordering | Ordered vs unordered processing of chunks | Alias-resolution errors |

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: the message broker, all pipeline services, the mock LLM, and Neo4j.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- The collected report corpus (text, source URL, download date) and the download script.
- Pipeline services: chunker, extraction workers, validator with its rules, alias normaliser, graph writer, dead-letter handling.
- Recorded LLM responses and the mock LLM service.
- Hand-labelled evaluation sample.
- Failure-injection and benchmark scripts, and Cypher queries checking duplicates and consistency.
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Graph statistics:** table of the number of nodes and edges per type, facts per report, and the share of rejected facts.
2. **Backpressure:** plot of throughput and consumer lag versus the number of workers, for several LLM rate limits.
3. **Duplicates:** table of duplicate nodes and edges for each combination of idempotency mechanisms under injected failures.
4. **Reprocessing:** table of rebuild time and the result of the consistency checks after a change of the validator rules.
5. **Validation and quality:** table of rejection reasons, and precision of facts on the labelled sample at each pipeline stage.

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of the message broker (Kafka or Dapr Pub/Sub) and Neo4j and the properties used in the project: delivery semantics, consumer groups and offsets, transactions and `MERGE`, uniqueness constraints.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- a new report ingested live, with its facts appearing in the graph together with their provenance,
- killing a worker during processing and showing that no duplicates are created,
- a rejected fact in the dead-letter queue with its reason.

## Practical Tips

- Ask the LLM for JSON output with a fixed schema (most APIs and Ollama support structured output) and reject anything that does not parse.
- Keep chunks small (e.g., a few paragraphs); long contexts are slower and less reliable.
- Inject failures by killing workers or by making the mock LLM return errors, then count duplicates with a Cypher query.

## References

- [Apache Kafka documentation](https://kafka.apache.org/documentation/): *Consumer groups*, *Delivery semantics*.
- [Neo4j Cypher manual](https://neo4j.com/docs/cypher-manual/current/): `MERGE`, constraints.
- [Ollama](https://ollama.com).
