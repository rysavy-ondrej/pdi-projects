# Project 1: Streaming Classification and Drift Detection of Encrypted Traffic

| | |
| --- | --- |
| **Main technologies** | Apache Kafka, Apache Flink (PyFlink) |
| **AI/ML component** | Pre-trained traffic classifier (e.g., LightGBM, exported to ONNX); drift detection (PSI) |
| **Data** | CESNET-TLS-Year22 (a reduced-size variant) |
| **Derived from** | Drift-Aware Detection of Unknown Encrypted Network Traffic (GJU) |

## Motivation

Most network traffic today is encrypted, so monitoring systems classify flows (e.g., "this flow belongs to Netflix" or "to Google Drive") using metadata only: packet sizes, directions, timing, and unencrypted TLS fields. In production this classification runs on a continuous stream. Flows arrive out of order, the traffic changes over time, and the system must survive failures without losing or duplicating results.

This project builds such a streaming pipeline and studies its behaviour as a **distributed stream-processing system**. The ML model is a fixed, simple component; the focus is on how Flink handles parallelism, state, event time, and failures.

## Key Concepts

- **Flow record:** a summary of one network connection (e.g., the sizes and timings of its first packets, byte counts, duration, TLS server name), with a label of the service it belongs to.
- **Event time vs processing time:** event time is when the flow actually happened (timestamp in the record); processing time is when Flink sees it. Results must be computed by event time, even when records arrive late or out of order.
- **Watermark:** Flink's estimate of "event time has progressed to T; no older events are expected". It decides when a window can be closed. Events arriving after the watermark are *late*.
- **Drift:** the traffic stops looking like the data the model was trained on. Encrypted traffic changes continuously: applications are updated, services move between CDNs, TLS libraries change, QUIC replaces TLS for some services, and traffic follows weekly and seasonal patterns. A classifier trained on early data silently loses accuracy. CESNET-TLS-Year22 spans a full year and shows this effect, which makes it suitable for this project.
- **Drift detection:** comparing statistics of a recent window of the stream with a reference window and raising an alert when they differ significantly. The model stays fixed; the goal is to *detect* drift, not to adapt to it.
- **PSI (Population Stability Index):** a single number measuring how far two histograms are apart. Bin a value (e.g., the predicted class) for the reference and the current window and compute `PSI = Σ (p_cur − p_ref) · ln(p_cur / p_ref)` over the bins. A common rule of thumb: PSI < 0.1 no change, 0.1–0.2 moderate change, > 0.2 significant drift.

## Research Question

Can a stream-processing pipeline classify network flows and detect distribution drift in real time with bounded latency, while producing correct results under out-of-order data and node failures?

## System Overview

```text
CESNET-TLS-Year22 ──► Replayer ──► Kafka topic "flows"
                                         │
                       ┌─────────────────▼─────────────────┐
                       │ Flink job (parallelism N)         │
                       │  1. parse, assign timestamps and  │
                       │     watermarks                    │
                       │  2. score flow with ONNX model    │
                       │  3. per-instance partial          │
                       │     histograms (sliding windows)  │
                       │  4. merge histograms, compute PSI │
                       └───────┬──────────────────┬────────┘
                               ▼                  ▼
                 Kafka "predictions"      Kafka "drift-alerts"
                               └────────┬─────────┘
                                        ▼
                              Dashboard (e.g., Grafana)
```

## Data

| Dataset | Content | Download |
| --- | --- | --- |
| CESNET-TLS-Year22 | About 507 million TLS flows from the CESNET backbone, 1 January – 31 December 2022, labelled with 180 web services | Python package [`cesnet-datazoo`](https://github.com/CESNET/cesnet-datazoo) ([documentation](https://cesnet.github.io/cesnet-datazoo/)); raw data on [Zenodo](https://zenodo.org/records/10608607) |

- Each record contains packet sequence features (sizes, directions, inter-arrival times of the first packets), flow statistics, the service label, and a timestamp. The features are already computed; you do not need to process PCAP files.
- The full dataset is very large. Download it with `cesnet-datazoo`, which offers reduced-size variants (the default `S` size has 25 million samples), and choose a size that your cluster can replay in minutes, not hours. Start the download early; it takes time.
- Dataset paper: *CESNET-TLS-Year22: A year-spanning TLS network traffic dataset from backbone lines*, Scientific Data, 2024, [doi:10.1038/s41597-024-03927-4](https://doi.org/10.1038/s41597-024-03927-4).

## Tasks

### Required Tasks

1. **Offline model.** Train a simple classifier (e.g., LightGBM) on the first two weeks of data (the *reference period*), export it to ONNX, and compute the reference histograms of the predicted classes and of the model confidence. Accuracy is not important; a reasonable baseline is enough.
2. **Replayer.** A Python script that reads flow records in timestamp order and publishes them to Kafka with a configurable speed-up factor (e.g., one day of traffic in one minute). It must be able to:
   - inject a controlled amount of out-of-order records (for the event-time experiment),
   - splice two distant periods together, e.g., week 1 directly followed by week 40 (*injected drift* with a known change point).
3. **Flink job** (PyFlink recommended):
   - assigns event-time timestamps and watermarks,
   - scores each flow with the ONNX model (e.g., with `onnxruntime` inside a map function) and writes predictions to Kafka,
   - computes drift statistics over sliding event-time windows (e.g., 1 day long, sliding every hour) and emits an alert to Kafka when PSI exceeds a threshold.
4. **Distributed drift computation.** A single global statistic over a parallel stream would force all data through one operator instance (parallelism 1). Because histograms can be merged, compute partial histograms in each parallel instance and merge them in a second, small stage. Implement both variants (single-stage and two-stage) so that they can be compared.
5. **Deployment** with Docker Compose or Kubernetes: Kafka, Flink JobManager, at least two TaskManagers.

### Optional Tasks

- A simple dashboard (e.g., Grafana) showing throughput, predictions per class, and PSI over time.
- Monitoring of the real error rate using the dataset labels with the **ADWIN** detector (e.g., from the [`river`](https://riverml.xyz) library). Real deployments do not have labels, but here it shows how well the label-free PSI signal tracks the actual accuracy loss.

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Scalability | Operator parallelism, number of TaskManagers | Throughput (flows/s), p50/p99 end-to-end latency | Does the pipeline scale with more resources, and where is the bottleneck (e.g., model scoring)? |
| Event time | Amount of out-of-order data, watermark delay, allowed lateness | Fraction of late/dropped events, change in PSI values | How much disorder can the pipeline tolerate before results become wrong? |
| Drift detection | Single-stage vs two-stage aggregation; parallelism; injected vs natural drift | Detection delay (in event time and wall-clock time), false alarms, throughput | Is drift detected, how fast, and what does the distributed design cost? |
| Fault tolerance | Checkpoint interval; kill a TaskManager mid-run; exactly-once vs at-least-once Kafka sink | Recovery time, duplicated/lost predictions and alerts, throughput cost | What does exactly-once cost, and what happens without it? |

**Measuring latency:** add the wall-clock time when the replayer sends a record and compare it with the time the result is read from the output topic.

**Drift ground truth:** use (1) the injected drift scenario with a known change point, and (2) a replay of the whole year, where natural drift is expected to grow over time.

### Optional Experiments

| Experiment | What to vary | What to measure |
| --- | --- | --- |
| State backend | HashMap (heap) vs RocksDB; drift window length | Throughput, memory use, checkpoint size and duration |

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: Kafka, a Flink JobManager, and at least two TaskManagers.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- Replayer with speed-up, out-of-order, and drift-injection options.
- Flink job source code.
- Model training script, the trained ONNX model, and the reference statistics.
- Benchmark and failure-injection scripts.
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Scalability:** plot of throughput and p50/p99 end-to-end latency versus parallelism (e.g., 1, 2, 4, 8) and number of TaskManagers, with the bottleneck operator identified.
2. **Event time:** table or plot of the fraction of late/dropped events and the deviation of PSI values versus the amount of out-of-order data and the watermark delay.
3. **Drift detection (injected):** plot of PSI over event time for the injected-drift scenario, with the change point and the alert marked; table of detection delay (event time and wall-clock), false alarms, and throughput for single-stage vs two-stage aggregation.
4. **Drift detection (natural):** plot of PSI over the whole year together with the model accuracy computed from the labels.
5. **Fault tolerance:** table of recovery time and the number of lost and duplicated predictions and alerts for exactly-once vs at-least-once sinks and different checkpoint intervals, including the throughput cost of exactly-once.

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of Apache Flink and Kafka and the properties used in the project: event time and watermarks, keyed state, checkpointing, Kafka partitions and transactions.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- the replay running with predictions and PSI values visible (dashboard or console),
- a drift alert triggered by the injected drift,
- killing a TaskManager and showing that the job recovers.

## Practical Tips

- Start with a small data size and parallelism 1; only scale up when the pipeline is correct.
- Model scoring in Python is usually the bottleneck. Score in mini-batches if per-record scoring is too slow.
- Keep the Kafka topic partition count at least as high as the maximum Flink parallelism you want to test.
- Repeat each measurement several times (at least 3) and report mean and variability.

## References

- [Apache Flink documentation](https://nightlies.apache.org/flink/flink-docs-stable/): sections *Timely Stream Processing*, *Watermarks*, *Checkpointing*, *Kafka connector*.
- [CESNET DataZoo](https://github.com/CESNET/cesnet-datazoo) and the paper *CESNET-TLS-Year22: A year-spanning TLS network traffic dataset from backbone lines*.
- [ONNX Runtime](https://onnxruntime.ai) and [`onnxmltools`](https://github.com/onnx/onnxmltools) for LightGBM export.
