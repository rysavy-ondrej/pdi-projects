# Project 5: Graph Database Benchmark for a Large CTI Knowledge Graph

| | |
| --- | --- |
| **Main technologies** | Neo4j and one distributed graph database (NebulaGraph recommended; JanusGraph on Cassandra possible) |
| **AI/ML component** | Text classifier predicting missing CWE links for CVEs (e.g., scikit-learn), stored as facts with confidence |
| **Data** | MITRE ATT&CK, CAPEC, CWE, NVD CVEs, CISA KEV |
| **Derived from** | LLM-Assisted Cyber Threat Intelligence Knowledge Graphs (PSUT) |

## Motivation

Cyber threat intelligence (CTI) naturally forms a graph: a vulnerability (CVE) affects products and is caused by a weakness (CWE); weaknesses enable attack patterns (CAPEC); attack patterns relate to techniques (ATT&CK) used by threat actors and malware. Analysts ask multi-hop questions such as "which attack techniques become possible through vulnerabilities in product X?".

The graph has hundreds of thousands of nodes, and every statement should keep its **provenance**: where it came from, how confident we are, and whether it was validated. Choosing a graph database and a data model for this workload is a distributed data management problem.

## Key Concepts

- **Property graph:** nodes and edges, both with properties. Queried with Cypher (Neo4j) or nGQL/openCypher (NebulaGraph) or Gremlin (JanusGraph).
- **Multi-hop traversal:** following several edges, e.g., CVE → CWE → CAPEC → ATT&CK technique. The cost grows quickly with the number of hops.
- **Provenance:** metadata about a fact (source, date, confidence, validation status). Two common models:
  - *edge properties:* stored directly on the edge, simple and compact;
  - *reified statements:* each fact becomes its own node linked to subject, object, and source; flexible (a fact can have several sources) but makes queries longer.
- **Distributed graph database:** data is split into partitions (shards) stored on several nodes. Traversals that cross partitions require network communication.

## Research Question

Which graph database and modelling choices scale best for a large, provenance-aware CTI knowledge graph?

## System Overview

```text
ATT&CK (STIX) ─┐
CAPEC, CWE    ─┤                       ┌──► Neo4j (single instance)
NVD CVEs      ─┼──► ETL (Python) ──► CSV ──┤
CISA KEV      ─┘        ▲              └──► NebulaGraph (cluster, ≥ 3 storage nodes)
                        │                          ▲
          CWE classifier (predicted links)         │
                                         Benchmark harness
                                         (query workload, metrics)
```

## Data

| Source | Content | Approx. size | Download |
| --- | --- | --- | --- |
| MITRE ATT&CK (Enterprise, STIX 2.1) | Techniques, tactics, groups, software, mitigations, and their relations | thousands of objects | [github.com/mitre-attack/attack-stix-data](https://github.com/mitre-attack/attack-stix-data) (file `enterprise-attack/enterprise-attack.json`) |
| CAPEC | Attack patterns with links to CWE and ATT&CK | about 600 patterns | [capec.mitre.org/data/downloads.html](https://capec.mitre.org/data/downloads.html) (XML) |
| CWE | Weaknesses and their hierarchy | about 1,000 weaknesses | [cwe.mitre.org/data/downloads.html](https://cwe.mitre.org/data/downloads.html) (XML) |
| NVD CVEs | All CVEs with descriptions, CWE, CVSS, and affected products (CPE) | about 250,000+ CVEs | Yearly JSON 2.0 feeds `https://nvd.nist.gov/feeds/json/cve/2.0/nvdcve-2.0-YYYY.json.gz` ([data feeds page](https://nvd.nist.gov/vuln/data-feeds)), or the [CVE API 2.0](https://nvd.nist.gov/developers/vulnerabilities) |
| CISA KEV | CVEs known to be exploited in the wild | about 1,700+ | [JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json), [CSV](https://www.cisa.gov/sites/default/files/csv/known_exploited_vulnerabilities.csv) ([catalog page](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)) |

- The CAPEC links (CWE → CAPEC → ATT&CK) are needed to connect CVEs to ATT&CK techniques.
- Simplify products: create vendor and product nodes from the CPE strings; do not import the full CPE dictionary.
- Prefer the yearly feeds (one file per year since 2002) over the API; the API is rate-limited and needs a free API key for reasonable speed.
- Record the download date and the versions of all sources (ATT&CK and CAPEC have version numbers) in the README; the sources change over time.

## Tasks

### Required Tasks

1. **Graph schema:** node types (CVE, CWE, CAPEC, Technique, Group, Software, Vendor, Product) and edge types; document it with a diagram.
2. **ETL:** Python scripts that download the sources and produce import files (CSV) for both databases.
3. **Two databases:** load the graph into Neo4j (single instance) and into a distributed database (NebulaGraph recommended) with at least three storage nodes.
4. **Two provenance models:** edge properties and reified statement nodes, using at least the source and confidence as provenance attributes. Both models are required in Neo4j; in the distributed database the edge-property model is sufficient.
5. **ML-predicted facts:** many CVEs have no specific CWE. Train a simple text classifier (e.g., TF-IDF + logistic regression) on CVE descriptions with known CWE, predict CWE for CVEs without one, and store the predictions as facts with `source = "model"`, a confidence value, and `validated = false`. Queries must be able to include or exclude them.
6. **Query workload:** a fixed set of 6–8 queries, for example:
   - 1–4-hop traversals ("ATT&CK techniques reachable from CVEs affecting product X"),
   - shortest path between two entities,
   - aggregations ("most frequent CWEs per vendor", "KEV CVEs per CWE"),
   - provenance filters ("only validated facts with confidence > 0.8").
7. **Benchmark harness:** runs the workload repeatedly (with warm-up) and records latency and throughput.

### Optional Tasks

- Synthetic growth of the graph (10× copies with new IDs) for a data-size experiment.
- A third database (e.g., ArangoDB or JanusGraph).

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Loading | Bulk import vs incremental inserts (batched transactions) | Load time, throughput (records/s) | How should the graph be loaded and kept up to date? |
| Query performance | Database, query type, hop count | p50/p99 latency, throughput with concurrent clients | Which database handles which query types better? |
| Data model (Neo4j) | Edge properties vs reified statements | Storage size, query latency, query complexity | What does flexible provenance cost? |
| Scaling (distributed DB) | Number of storage nodes, number of partitions, replication factor | Query latency, load time | Does adding nodes help, and for which queries? |

### Optional Experiments

| Experiment | What to vary | What to measure |
| --- | --- | --- |
| Data size | Original vs 10× synthetic growth | Latency growth, memory use |

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: Neo4j and the distributed graph database with at least three storage nodes.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- Download and ETL scripts for all sources (not the downloaded data).
- Graph schema with a diagram.
- CWE classifier training and prediction script.
- Query workload: each query in the query language of each database, with its parameters.
- Benchmark harness and the collected results.
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Graph statistics:** table of the number of nodes and edges per type in each database, including the number of ML-predicted facts and their confidence distribution.
2. **Loading:** table of load time and throughput for bulk import and incremental inserts, per database.
3. **Query performance:** table or plot of p50/p99 latency per query and database, and a plot of throughput versus the number of concurrent clients.
4. **Data model:** table of storage size and query latency for edge properties vs reified statements (Neo4j).
5. **Scaling:** plot of query latency and load time versus the number of storage nodes, partitions, and the replication factor.
6. **CWE classifier:** accuracy on held-out CVEs with a known CWE (one table, briefly).
7. **Recommendations:** which database and data model to use for which kind of query, supported by the results.

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of the graph databases used and their relevant properties: storage and partitioning model, replication, how multi-hop traversals are executed across partitions, bulk loading.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- a multi-hop query (CVE → CWE → CAPEC → ATT&CK) in both databases,
- a provenance filter that includes and excludes the ML-predicted facts.

## Practical Tips

- Neo4j Community Edition does not support clustering; this is fine. Neo4j serves as the single-node reference, and the distributed database satisfies the multi-node requirement.
- Load a small subset first (e.g., CVEs from one year) and verify queries before loading everything.
- Run the benchmark client in a separate container, use the same query parameters for all databases, and separate cold-cache and warm-cache runs.
- Check that both databases return the same results for each query; a fast wrong answer is not a result.

## References

- [Neo4j documentation](https://neo4j.com/docs/), [NebulaGraph documentation](https://docs.nebula-graph.io), [JanusGraph](https://janusgraph.org).
- [MITRE ATT&CK STIX data](https://github.com/mitre-attack/attack-stix-data).
- [NVD developer documentation](https://nvd.nist.gov/developers).
