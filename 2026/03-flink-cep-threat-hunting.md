# Project 3: Complex Event Processing for Network Threat-Hunting Patterns

| | |
| --- | --- |
| **Main technologies** | Apache Kafka, Apache Flink (Flink SQL with `MATCH_RECOGNIZE`, Flink CEP library), Apache Spark (batch reference) |
| **AI/ML component** | Pre-trained domain classifier (e.g., scikit-learn, detecting algorithmically generated domains) used inside a pattern |
| **Data** | Zeek logs (conn, dns, ssl) from public PCAPs with malware traffic |
| **Languages** | Python for most parts; Java for the Flink CEP library (not available in PyFlink) |
| **Derived from** | Evidence-Grounded Natural-Language Assistant for Network Threat Hunting (GJU) |

## Motivation

Threat hunters search network telemetry for behavioural patterns: a host contacting the same server at regular intervals (beaconing of malware to its command-and-control server), a DNS lookup of a suspicious domain followed by a TLS connection to it, or one host probing many addresses or ports (scanning).

Stream processing engines can detect such patterns continuously, as events arrive, instead of in nightly batch jobs. To do so they must keep per-host state, cope with late and out-of-order events, and scale with the event rate. This project implements a few hunting patterns in Flink, checks their correctness against a batch implementation, and studies their state and performance.

## Key Concepts

- **Zeek logs:** Zeek is a network monitor that turns packet captures into structured logs: `conn.log` (one record per connection), `dns.log` (DNS queries and answers), `ssl.log` (TLS handshakes, including the server name).
- **Complex event processing (CEP):** detecting patterns over sequences of events, e.g., "event A followed by event B for the same host within 1 minute".
- **`MATCH_RECOGNIZE`:** an SQL clause for row-pattern matching, supported by Flink SQL. A pattern is written like a regular expression over events (`PATTERN (dns tls)`), with conditions in `DEFINE`.
- **Flink CEP library:** a Java API for the same purpose, more flexible than SQL.
- **Windowed aggregation:** some patterns are not sequences but statistics over a time window (e.g., number of distinct destination ports per source in 1 minute). These are better expressed with ordinary windowed `GROUP BY`.
- **DGA domains:** malware often uses domain generation algorithms that produce random-looking names (e.g., `xjw3kq9tz.com`). A simple character-based classifier can score how likely a domain is generated.

## Research Question

How well can stream processing engines express and scale stateful threat-hunting patterns over network telemetry, and how do their results compare to batch analysis?

## System Overview

```text
PCAPs ──► Zeek ──► conn/dns/ssl logs ──► Replayer ──► Kafka topics
                         │                                 │
                         ▼                                 ▼
               Spark batch job                  Flink jobs (SQL and CEP)
               (reference detections)           with DGA classifier
                         │                                 │
                         └──────────► Comparison ◄─────────┘
                                   (precision / recall)
```

## Data

| Dataset | Content | Download |
| --- | --- | --- |
| CTU-13 | 13 botnet scenarios (e.g., Neris, Rbot, Murlo); botnet-only PCAPs and labelled flows | [Dataset page](https://www.stratosphereips.org/datasets-ctu13); full archive (1.9 GB) at `https://mcfp.felk.cvut.cz/publicDatasets/CTU-13-Dataset/CTU-13-Dataset.tar.bz2` |
| Stratosphere normal captures | Benign traffic of individual users (several captures include PCAP and Zeek/Bro logs, e.g., CTU-Normal-7, CTU-Normal-10) | [Dataset page](https://www.stratosphereips.org/datasets-normal) |
| malware-traffic-analysis.net (optional) | PCAPs of individual malware infections with descriptions | [malware-traffic-analysis.net](https://www.malware-traffic-analysis.net) |
| Tranco list | Ranking of popular (benign) domains, used as negative examples for the DGA classifier | [tranco-list.eu](https://tranco-list.eu) |
| DGA implementations | Open-source reimplementations of many malware DGAs, used to generate positive examples | [github.com/baderj/domain_generation_algorithms](https://github.com/baderj/domain_generation_algorithms) |

- Choose 3–5 malicious captures that contain the behaviour of your patterns (e.g., a bot with periodic C2 connections, a scanning bot) and 2–3 benign captures, so that patterns also have something to *not* match.
- CTU-13 provides botnet-only PCAPs; generate the Zeek logs yourself with the [Zeek Docker image](https://hub.docker.com/r/zeek/zeek) (`zeek -r capture.pcap`). Where Zeek logs are already provided, you may use them.
- Captures come from different times; shift timestamps when merging them into one stream and document how.
- Malware PCAPs and samples may trigger antivirus software; work with them only inside containers or virtual machines.

## Tasks

### Required Tasks

1. **Log preparation and replayer:** produce Zeek logs and a replayer that publishes them to Kafka in timestamp order with a configurable speed-up and an option to inject out-of-order and late events.
2. **DGA classifier:** train a simple model offline (e.g., logistic regression on character n-grams in scikit-learn) and use it inside the stream to score DNS queries. Accuracy is not the goal; a working baseline is enough.
3. **Three hunting patterns:**
   - **Beaconing:** many connections from one host to one destination with regular inter-arrival times (e.g., low standard deviation of the intervals within a window). Implement as windowed aggregation.
   - **DNS → TLS to a suspicious domain:** a DNS query with a high DGA score followed by a TLS connection to the resolved address within a short time. Implement as a sequence pattern.
   - **Scanning:** one source contacting many distinct hosts (horizontal) or ports (vertical) within a window. Implement as windowed aggregation.
4. **Two implementations in Flink:**
   - all three patterns in **Flink SQL** (`MATCH_RECOGNIZE` for sequences, windowed aggregation for the others),
   - the sequence pattern (DNS → TLS) also in the **Flink CEP library** (Java).
5. **Batch reference:** the same three patterns as a Spark SQL batch job over the complete logs. Its output serves as the ground truth for streaming results.
6. **Deployment** with Docker Compose or Kubernetes: Kafka, Flink JobManager, at least two TaskManagers.

### Optional Tasks

- A fourth pattern, e.g., a large outbound transfer after a connection to a rarely contacted destination.
- All patterns also in the Flink CEP library.

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Correctness | Flink SQL vs Flink CEP vs batch reference | Precision and recall of detections against the batch reference | Do streaming implementations find the same incidents as batch analysis? |
| Disorder | Amount of out-of-order and late events, watermark delay | Missed and duplicated detections, detection delay | How sensitive are the patterns to event-time problems? |
| Performance | Input rate, parallelism | Throughput, detection latency (event arrival → alert) | How do the patterns scale, and which one is the most expensive? |
| State | Pattern time window length | State size, checkpoint size and duration | How does window length affect state and checkpointing cost? |

Also include a short qualitative comparison of SQL and CEP (lines of code, what could or could not be expressed, debugging experience).

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: Kafka, a Flink JobManager, and at least two TaskManagers.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- Download scripts for the captures and scripts generating the Zeek logs (not the PCAPs themselves).
- Log replayer with out-of-order and late-event options.
- DGA classifier training script and the trained model.
- Flink SQL and Flink CEP implementations of the patterns.
- Spark batch reference job and the comparison script.
- List of expected detections for the chosen captures (CSV).
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Correctness:** table with, for each pattern, the number of detections of the batch reference, Flink SQL, and Flink CEP, and precision and recall against the batch reference, with an explanation of the differences.
2. **Disorder:** plot of missed and duplicated detections versus the amount of out-of-order data and the watermark delay.
3. **Performance:** plot of throughput and detection latency versus input rate and parallelism, for each pattern.
4. **State:** plot or table of state size and checkpoint size and duration versus the pattern window length.
5. **SQL vs CEP:** table comparing lines of code, what could and could not be expressed, and observed limitations.
6. **DGA classifier:** accuracy on a held-out test set (one table, briefly).

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of Apache Flink and Kafka and the properties used in the project: event time and watermarks, keyed state for pattern matching, `MATCH_RECOGNIZE` and the CEP library, checkpointing.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- a capture replayed into Kafka, with detections of each pattern appearing in real time,
- one detection (e.g., beaconing) traced back to the underlying Zeek events.

## Practical Tips

- Start with one pattern end to end (replayer → Flink SQL → output topic → comparison with batch) before adding more.
- Beaconing is hard to express with `MATCH_RECOGNIZE`; windowed aggregation with interval statistics is simpler and acceptable.
- Linking DNS and TLS events needs a common key (e.g., client IP and resolved server IP); think about this before writing the pattern.
- Small differences between streaming and batch results are expected (window boundaries, late events). Explain them; do not hide them.

## References

- [Apache Flink documentation](https://nightlies.apache.org/flink/flink-docs-stable/): *Pattern Recognition* (`MATCH_RECOGNIZE`), *FlinkCEP*, *Windowing TVF*, *Watermarks*.
- [Zeek documentation](https://docs.zeek.org).
- [Stratosphere IPS datasets](https://www.stratosphereips.org/datasets-overview).
