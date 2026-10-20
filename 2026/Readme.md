
# 2026 Course Projects

Each project demonstrates and evaluates a distributed computing or storage technology on a security-related problem, with a small AI/ML component.

| # | Project | Main technologies | AI/ML component |
| --- | --- | --- | --- |
| 1 | [Streaming Classification and Drift Detection of Encrypted Traffic](01-flink-streaming-traffic-drift.md) | Kafka, Flink | Traffic classifier (ONNX), drift detection |
| 2 | [Time-Sliced Traffic Analytics on Spark at Year Scale](02-spark-year-scale-traffic-analytics.md) | Spark, Parquet | MLlib classifier, drift statistics |
| 3 | [Complex Event Processing for Network Threat-Hunting Patterns](03-flink-cep-threat-hunting.md) | Kafka, Flink SQL/CEP, Spark | Domain (DGA) classifier |
| 4 | [Fault-Tolerant Multi-Agent Alert Triage on Dapr](04-dapr-multi-agent-alert-triage.md) | Dapr Workflow, Pub/Sub, State | LLM agents |
| 5 | [Graph Database Benchmark for a Large CTI Knowledge Graph](05-graph-db-cti-benchmark.md) | Neo4j, NebulaGraph | CWE text classifier |
| 6 | [Stream Ingestion of LLM-Extracted Facts into a Knowledge Graph](06-streaming-kg-ingestion.md) | Kafka or Dapr, Neo4j | LLM information extraction |
| 7 | [Distributed Similarity Graph and Malware Clustering](07-spark-malware-similarity-graph.md) | Spark MLlib LSH, GraphFrames | LSH, graph clustering |
| 8 | [Consistent Pseudonymization in a Parallel Log Stream](08-consistent-pseudonymization-stream.md) | Kafka, Flink or Dapr, Redis | NER for personal data |

Every project description has the same structure: motivation, key concepts, research question, system overview, data, required and optional tasks, experiments, expected output, practical tips, and references.
