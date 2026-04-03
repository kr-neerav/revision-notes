# REVISION NOTES: DATA QUALITY AT SCALE

## LAYER 1: The Dimensions of Data Quality
* **Core Concept:** "Good data" does not exist in a vacuum; it is only good if it is fit for its intended downstream purpose.
* **The Six Dimensions:**
    1.  **Validity:** Conforms to required format, type, and range (e.g., integer vs. string).
    2.  **Completeness:** All expected records/fields are present (no dropped events).
    3.  **Consistency:** Agrees with itself across different systems.
    4.  **Accuracy:** Reflects the real-world state or event (e.g., correct timestamp, correct currency unit).
    5.  **Timeliness:** Available when the consumer needs it.
    6.  **Uniqueness:** Exactly one record per real-world entity (no duplicates).
* **Production Impact:** Basic schema validation only catches Validity/Completeness. You must engineer tripwires for Accuracy and Uniqueness to prevent business logic failures (like overbilling).

## LAYER 2: Ingestion & Schema Validation
* **Core Concept:** Catch malformed data at the front door before it enters your infrastructure ("Schema-on-Write").
* **Key Patterns:**
    * **Strict Typing (Schema Registry):** Force producers to serialize data using enforced protocols (Protobuf, Avro, JSON Schema).
    * **Dead Letter Queue (DLQ):** Route invalid events to a separate storage location instead of dropping them. Preserves completeness and allows for operational replay once the upstream bug is patched.
* **Hybrid Validation Approaches (Balancing Velocity and Safety):**
    * *Medallion Architecture:* Ingest raw to Bronze (Schema-on-Read), enforce strict schema moving to Silver (Schema-on-Write).
    * *Envelope Pattern:* Strict validation on routing keys, flexible payload for properties.
    * *Audit Mode:* Ingest invalid data but flag it with metadata and fire high-priority alerts.

## LAYER 3: Pipeline Circuit Breakers
* **Core Concept:** A valid schema does not guarantee valid business logic. Stop bad data from overwriting good data during transformations.
* **Write-Audit-Publish (WAP) Pattern:**
    1.  **Write:** Run transformations and write to a hidden staging area/branch.
    2.  **Audit:** Run semantic data quality checks against the hidden branch (e.g., checks for zero-values, fan-outs, moving average deviations).
    3.  **Publish:** Perform an atomic swap to make the staging area production-visible ONLY if the audit passes.
* **Key Tradeoff:** When a circuit breaker trips, you actively choose *Timeliness* over *Accuracy* (stale data is better than catastrophically wrong data).

## LAYER 4: Data Observability & Anomaly Detection
* **Core Concept:** Static rules do not scale to thousands of tables and fail when business baselines shift naturally (e.g., holidays, growth).
* **Modern Observability Components:**
    * **Automated Profiling:** System continuously builds statistical profiles of every column (null %, distinct counts, min/max).
    * **Time-Series Anomaly Detection:** Machine learning predicts what the metric *should* be based on historical seasonality, flagging statistical deviations rather than hard limits.
    * **Data Lineage:** Maps dependencies. When an anomaly triggers, lineage traces the failure upstream to the exact source (e.g., a specific Kafka topic), drastically reducing Mean Time To Resolution (MTTR).

## LAYER 5: Data Contracts & Organizational Ownership
* **Core Concept:** Data quality is ultimately an organizational problem caused by decoupled producers and consumers. Data must be treated as a product, not an exhaust pipe.
* **Data Contracts:** Formal, version-controlled agreements (usually YAML) defining schema, semantics, and SLAs between software engineers and data teams.
* **CI/CD Enforcement:** If an upstream software engineer opens a PR that breaks a downstream consumer's contract (e.g., dropping a required column), the CI pipeline fails the build.
* **Shift-Left Ownership:** Forces the producer to coordinate with the consumer before making breaking changes, shifting accountability for data quality to the source.