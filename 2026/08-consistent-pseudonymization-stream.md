# Project 8: Consistent Pseudonymization in a Parallel Log Stream

| | |
| --- | --- |
| **Main technologies** | Apache Kafka, and either Apache Flink (PyFlink) or Dapr services; Redis or a Dapr State Store; a secret store |
| **AI/ML component** | Pre-trained named-entity recognition (NER) model detecting personal data in free-text log messages (e.g., Microsoft Presidio or a Hugging Face NER model) |
| **Data** | Synthetic authentication and system logs from a documented generator |
| **Derived from** | A Modular Masking Engine for Privacy-Preserving Autonomous Threat Analysis (PSUT) |

## Motivation

Security logs contain personal data: IP addresses, usernames, hostnames, sometimes names or e-mail addresses in free-text messages. Before logs are shared with analysts or external tools, this data should be masked. To keep the logs useful, masking must be **consistent**: the same entity must always get the same pseudonym, so that sequences such as "20 failed logins from one IP" remain detectable.

In a parallel stream-processing system this becomes a **distributed consistency problem**: many operator instances must agree on pseudonyms, across restarts and rescaling, without leaking real identities.

## Key Concepts

- **Pseudonymization:** replacing an identifier with a substitute (pseudonym), e.g., `alice` → `user_7f3a9c`. Unlike anonymization, the mapping may be reversible by authorised people.
- **Strategy A, deterministic (keyed HMAC):** `pseudonym = HMAC(secret_key, value)`. Every instance with the same key computes the same pseudonym, with no shared state. Reversal is impossible without a lookup table.
- **Strategy B, shared vault:** a mapping table `value → random pseudonym` stored in Redis or a Dapr state store, encrypted with a key from a secret store. It allows controlled reversal (de-pseudonymization), but all instances depend on the shared store.
- **First-write race:** in Strategy B, two instances may see a new entity at the same time and each create a different pseudonym. This must be prevented, e.g., by an atomic set-if-absent (`SET NX` in Redis), by optimistic concurrency (ETags in the Dapr state store), or by partitioning the stream by entity so that one instance owns each entity.
- **Canary values:** artificial identities planted in the input (e.g., the username `canary_user_42`). If any canary appears in the output, masking leaked data.
- **NER (named-entity recognition):** an ML model that finds names, e-mail addresses, IPs, and similar entities in free text. Structured fields can be masked by rules, but free-text messages need NER.

## Research Question

How can pseudonyms be kept consistent across parallel stream-processing instances without leaking identities, and what is the performance cost of each approach?

## System Overview

```text
Log generator ──► Kafka "raw-logs"
                        │
          ┌─────────────▼───────────────┐
          │ Masking operator (×N)       │       Secret store (key)
          │  • structured fields: rules │◄────────────┘
          │  • free text: NER model     │
          │  • Strategy A (HMAC) or     │◄──► Vault (Redis / Dapr state)
          │    Strategy B (vault)       │       for Strategy B
          └─────────────┬───────────────┘
                        ▼
                Kafka "masked-logs" ──► Detector (brute force, scanning)
                        │
                        ▼
                 Leakage checker (canaries)
```

## Data

A **log generator** (Python) that you write and document. It produces authentication and system logs for a simulated organisation (e.g., 500 users, 200 hosts) with:

- benign activity (logins, logouts, occasional failed logins),
- attacks: brute force (many failures for one user), password spraying (one password tried on many users), port scanning,
- personal data in structured fields (source IP, username, hostname) **and** in free-text messages (e.g., `"Password reset requested by Alice Novak <alice.novak@example.org>"`),
- planted canary values in both structured and free-text fields,
- ground-truth labels of the attacks, so that the detector can be evaluated.

No external dataset is required. Use the following to make the generated data realistic:

| Resource | Use | Link |
| --- | --- | --- |
| Faker (Python library) | Realistic names, usernames, e-mail addresses, hostnames, IP addresses | [faker.readthedocs.io](https://faker.readthedocs.io) |
| Microsoft Presidio | Pre-trained PII detection for free-text fields (the AI/ML component); also usable to check the generator's free-text output | [microsoft.github.io/presidio](https://microsoft.github.io/presidio/) |

- Fix the random seed of the generator, and include the generator configuration in the repository, so that every experiment uses exactly the same input.
- Generated logs are synthetic, so no real personal data is processed. Real logs must not be used.

## Tasks

### Required Tasks

1. **Log generator** as described above, with a configurable rate and number of distinct entities.
2. **Masking operator** in one platform of your choice (Flink or Dapr), running with parallelism of at least 2:
   - rule-based masking of structured fields (IP, username, hostname),
   - NER-based detection and masking of personal data in free-text fields,
   - **Strategy A** (keyed HMAC) and **Strategy B** (encrypted shared vault with a safe solution of the first-write race).
3. **Key handling:** the HMAC key and the vault encryption key are loaded from a secret store, never from code or configuration files.
4. **De-pseudonymization** (Strategy B): a small service that reverses a pseudonym for an authorised request and writes each request to an access log.
5. **Detector** on masked data: windowed rules for brute force, password spraying, and scanning.
6. **Leakage checker:** scans the output for canary values and other known identities.

### Optional Tasks

- Key rotation during processing and a strategy for keeping sequences linkable across the rotation.
- Prefix-preserving IP pseudonymization (e.g., Crypto-PAn), so that subnet-based rules keep working.
- Implementation in both Flink and Dapr.

## Experiments

### Required Experiments

| Experiment | What to vary | What to measure | Question it answers |
| --- | --- | --- | --- |
| Overhead | No masking vs Strategy A vs Strategy B; with/without NER | Throughput, p50/p99 latency | What does each masking step cost? Is NER the bottleneck? |
| Consistency | Parallelism, restarts, rescaling; Strategy B with and without race protection | Number of entities with more than one pseudonym | Do all instances agree on pseudonyms in every situation? |
| Leakage | Rules only vs rules + NER | Canaries and identities found in the output (per field type) | What leaks, and does NER stop it? |
| Utility | Masked vs unmasked input | Detection rate and false alarms of the detector | Does masking change the detection results? |

### Optional Experiments

| Experiment | What to vary | What to measure |
| --- | --- | --- |
| Vault contention | Load, number of distinct entities, number of instances | Vault latency, hotspots |
| Key rotation | Rotate the key during processing | Broken sequences, recovery approach |

## Expected Output

### 1. Git Repository

- `README.md` with instructions to deploy the system and to reproduce every experiment and plot with a few commands.
- Deployment files (Docker Compose or Kubernetes) that start the whole system: Kafka, the masking operator with parallelism of at least 2, the vault, the secret store, the detector, and the de-pseudonymization service.
- Data download and preparation scripts, with the dataset versions and download dates. Do not commit large datasets.
- Log generator with its configuration and random seed, and its documentation.
- Masking operator with Strategy A, Strategy B, and NER-based masking of free text.
- Secret-store configuration (without real secrets).
- De-pseudonymization service with its access log.
- Detector, leakage checker, and benchmark scripts.
- Experiment scripts and plotting scripts that produce all results listed below from the raw measurements.

### 2. Experimental Results

The following results are required. Repeat each measurement at least three times and report the mean and variability (e.g., standard deviation or min–max). State the hardware and configuration used.

1. **Overhead:** table or plot of throughput and p50/p99 latency for no masking, Strategy A, and Strategy B, each with and without NER.
2. **Consistency:** table of the number of entities with more than one pseudonym for each scenario (parallelism, restart, rescaling; Strategy B with and without race protection).
3. **Leakage:** table of leaked canaries and identities per field type, for rules only and rules + NER.
4. **Utility:** table of detection rate and false alarms of the detector on masked and unmasked input.
5. **Recommendations:** when to use Strategy A and when Strategy B, supported by the results.

### 3. Technical Report (8–12 pages)

- Problem description and research question.
- System architecture with a diagram.
- Description of the chosen platform (Flink or Dapr) and the properties used in the project: parallelism and partitioning, state, concurrency control in Redis or the Dapr state store, secret management.
- Experiment design: what is varied, what is measured, and why.
- Results listed above, with interpretation: which configuration is better, and why.
- Answer to the research question.
- Limitations and possible improvements.

### 4. Presentation and Live Demo (about 15 minutes)

A short presentation of the architecture and the main results, and a live demo showing:

- a log line before and after masking, including a masked free-text field,
- the same user receiving the same pseudonym in different parallel instances,
- an authorised de-pseudonymization with its access-log entry,
- the leakage checker output.

## Practical Tips

- With Strategy A, consistency is guaranteed by design; the interesting results are in Strategy B and in the NER part. Plan your time accordingly.
- NER is much slower than rule-based masking. Run it only on free-text fields, and consider batching.
- To demonstrate the first-write race, generate many new entities at the same time and run several instances without partitioning by entity.
- Never print real secrets in logs; check this as part of the leakage test.

## References

- [Microsoft Presidio](https://microsoft.github.io/presidio/).
- [Apache Flink documentation](https://nightlies.apache.org/flink/flink-docs-stable/) or [Dapr documentation](https://docs.dapr.io): *State management*, *Secrets*.
- [Redis `SET` with `NX` option](https://redis.io/docs/latest/commands/set/).
