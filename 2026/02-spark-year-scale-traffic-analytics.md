# Project 2: Time-Sliced Traffic Analytics on Spark at Year Scale

| | |
| --- | --- |
| **Main technologies** | Apache Spark (PySpark, Spark SQL, MLlib), Parquet (optionally Delta Lake) |
| **AI/ML component** | Traffic classifier trained with Spark MLlib (e.g., random forest); drift statistics (PSI) |
| **Data** | CESNET-TLS-Year22 (a reduced-size variant) |
| **Derived from** | Drift-Aware Detection of Unknown Encrypted Network Traffic (GJU) |

## Motivation

Network traffic changes over time, so a traffic classifier must be evaluated realistically: train on earlier data and test on later data, and repeat this over many time slices of a year-long dataset. This produces a large batch-processing workload whose cost depends heavily on how the data is stored (file format, partitioning, file sizes) and how the cluster is configured.

This project treats the evaluation as a **big-data engineering problem**: the classifier is simple and fixed; the focus is on data layout, partitioning, shuffles, and scalability of Spark jobs.

## Key Concepts

- **Columnar format (Parquet):** stores data column by column with compression and statistics, so queries read only the columns and row groups they need.
- **Partitioning:** splitting a dataset into directories by a column value (e.g., `date=2022-03-14/`). Queries that filter on that column skip the other directories (*partition pruning*). Too many small partitions, however, create many small files and tasks.
- **Shuffle:** redistribution of data between executors (e.g., for `groupBy` or `join`). It is usually the most expensive part of a Spark job.
- **Adaptive Query Execution (AQE):** Spark's runtime optimisation that changes the number of shuffle partitions and join strategies based on actual data sizes.
- **Strong vs weak scaling:** strong scaling keeps the data size fixed and adds executors; weak scaling increases the data size together with the executors.
- **Time-sliced evaluation:** train on months 1..k, test on month k+1, for k = 1..11. The drop in accuracy over time shows *drift*.
- **PSI (Population Stability Index):** a number measuring how much a distribution (e.g., of a feature) changed compared with a reference period. PSI > 0.2 is a common threshold for significant drift.

## Research Question

How do data layout and cluster configuration affect the cost and scalability of a time-based train/test evaluation over a year of network traffic?

## System Overview

```text
CESNET-TLS-Year22 ──► Ingestion job ──► Parquet (partitioned)
                                              │
                    ┌─────────────────────────┼──────────────────────────┐
                    ▼                         ▼                          ▼
          Weekly statistics job     Time-sliced evaluation job   Benchmark queries
          (feature histograms,      (train on 1..k, test on k+1,
           PSI vs reference)         MLlib classifier)
                    └─────────────────────────┬──────────────────────────┘
                                              ▼
                                Results (Parquet/CSV) ──► plots
```

## Data

| Dataset | Content | Download |
| --- | --- | --- |
| CESNET-TLS-Year22 | About 507 million TLS flows from the CESNET backbone, 1 January – 31 December 2022, labelled with 180 web services | Python package [`cesnet-datazoo`](https://github.com/CESNET/cesnet-datazoo) ([documentation](https://cesnet.github.io/cesnet-datazoo/)); raw data on [Zenodo](https://zenodo.org/records/10608607) |

- Each record contains packet sequence features, flow statistics, the service label, and a timestamp.
- `cesnet-datazoo` offers reduced-size variants (the default `S` size has 25 million samples). Pick a variant large enough that Spark jobs take minutes (so that differences are measurable) but small enough to fit your cluster. Start the download early; it takes time.
- Dataset paper: *CESNET-TLS-Year22: A year-spanning TLS network traffic dataset from backbone lines*, Scientific Data, 2024, [doi:10.1038/s41597-024-03927-4](https://doi.org/10.1038/s41597-024-03927-4).
- For the scaling experiments you will need several data sizes; produce them by sampling the chosen variant (e.g., 10 %, 25 %, 50 %, 100 %).

## Tasks

### Required Tasks

1. **Ingestion:** convert the dataset into Parquet with an explicit schema. Produce at least two layouts: partitioned by date and not partitioned.
2. **Weekly statistics:** for a small set of features (e.g., 5–10, plus the distribution of service labels), compute per-week histograms and the PSI of each week against a reference period (e.g., the first two weeks).
3. **Time-sliced evaluation:** for each month k, train a Spark MLlib classifier (e.g., random forest with fixed hyperparameters) on months 1..k and test on month k+1. Record accuracy or macro-F1 for each slice.
4. **Report output:** a plot of PSI and accuracy over the year, showing how drift relates to the drop in accuracy.
5. **Deployment:** a Spark cluster with at least one master and two workers (Docker Compose standalone cluster or Kubernetes).
6. **Benchmark harness:** a script that runs each job with given settings several times and collects runtime, shuffle size, spill, and number of tasks from the Spark event logs or REST API.

### Optional Tasks

- Delta Lake tables with compaction and Z-ordering.
- An incremental variant of the evaluation that reuses statistics computed in previous slices instead of recomputing them.

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Data layout | Partitioned by date vs none; file size (few large vs many small files) | Query and job runtime, data read, number of tasks | How much does layout matter for time-based queries? |
| Scaling | 1..N executors (strong scaling); data size together with executors (weak scaling) | Runtime, speed-up, parallel efficiency | Does the evaluation scale, and where does it stop scaling? |
| Shuffle | AQE on/off, number of shuffle partitions | Shuffle volume, spill, runtime | How should shuffle be configured for this workload? |

### Optional Experiments

| Experiment | What to vary | What to measure |
| --- | --- | --- |
| Delta layout | Compaction, Z-ordering | Job runtime, files read |
| Incremental vs full | Reusing previous results vs full recomputation per slice | Runtime |

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: a Spark master and at least two workers.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- Ingestion job producing all evaluated data layouts.
- Weekly statistics job and time-sliced evaluation job.
- Benchmark harness and the collected Spark metrics (exported as CSV).
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Drift over the year:** one figure with the weekly PSI of the selected features and the monthly accuracy or macro-F1 of the time-sliced evaluation.
2. **Data layout:** table of runtime, data read, number of files, and number of tasks for each layout, for at least three representative queries or jobs.
3. **Scaling:** strong-scaling plot (runtime and speed-up versus number of executors, with the ideal speed-up line) and weak-scaling plot (parallel efficiency).
4. **Shuffle:** table of shuffle volume, spill, and runtime with AQE on and off and for different numbers of shuffle partitions.
5. **Recommendations:** a short list of data-layout and configuration recommendations, each supported by a measured result.

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of Apache Spark and Parquet and the properties used in the project: the execution model (jobs, stages, tasks), partitioning and partition pruning, shuffle, and Adaptive Query Execution.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- the time-sliced evaluation running on the cluster, with the Spark UI showing stages and shuffle,
- the same query on the partitioned and the unpartitioned layout, with the difference in data read.

## Practical Tips

- When running several workers on one machine, limit cores and memory per worker; otherwise "more executors" will not mean more resources and scaling results will be misleading. Use lab machines or a cloud cluster for the scaling experiment if possible.
- Clear caches between runs (or restart the application) so that runs are comparable.
- The Spark UI and event logs (`spark.eventLog.enabled=true`) contain all metrics you need; do not measure only wall-clock time.
- Keep the model and its hyperparameters fixed across all experiments.

## References

- [Apache Spark documentation](https://spark.apache.org/docs/latest/): *SQL Performance Tuning*, *Adaptive Query Execution*, *MLlib*, *Monitoring*.
- [Delta Lake](https://delta.io).
- [CESNET DataZoo](https://github.com/CESNET/cesnet-datazoo) and the paper *CESNET-TLS-Year22: A year-spanning TLS network traffic dataset from backbone lines*.
