# Staff-Level Architecture Notes: Real-Time Telemetry & Sub-Second Dashboard Pipeline
*Comprehensive Revision Notes: Macro Data Flow, Stream Deduplication, Storage Physics, and Idempotent Orchestration*

## 1. Event-Driven Signaling & Downstream Triggers
* **The Goal:** Notify downstream OLAP consumers (ClickHouse) to rebuild materialized views and dashboards when upstream Spark backfills or pipeline micro-batches complete.
* **The Anti-Pattern (Hardcoded Alerts):** Hardcoding manual alerts at the end of Spark scripts or forcing downstream systems to execute expensive full-table scans to detect changes.
* **The Staff-Level Fix (Table Format Native Features):** Leverage Apache Iceberg’s commit history and snapshot diffs. The downstream orchestrator or database listens for new metadata commits to identify exact changed partitions, enabling efficient, incremental downstream refreshes.

## 2. The Macro Data Flow & Data Quality
* **The Execution Flow:** 1. Telemetry is fired and sent to a local Sidecar.
  2. Batched and sent to a Central Collector.
  3. Pushed into Kafka (with at-least-once delivery enabled).
  4. Consumed by Spark Structured Streaming (15-minute micro-batches).
  5. Merged into Apache Iceberg (Data Lake).
  6. Synced incrementally to ClickHouse for sub-second dashboard rendering.
* **The Constraint (Schema Chaos):** 14 different product teams frequently evolve their telemetry schemas.
* **The Anti-Pattern:** Baking data validation rules directly into the Spark code, creating a hardcoded bottleneck that requires deployment for every schema change.
* **The Staff-Level Fix (Config-Driven Validation):** Extract data quality rules into a separate configuration registry and validation library. Spark imports this library to evaluate rules on the fly, keeping the streaming job completely decoupled from schema evolution.

## 3. Stream Deduplication & Storage Physics
* **The Threat:** Kafka's at-least-once delivery guarantees duplicate events. Relying on a nightly batch job to clean duplicates inflates real-time dashboard metrics all day long.
* **The Solution (Micro-Batch Upserts):** Spark Structured Streaming executes a `MERGE` operation (upsert) directly into Iceberg every 15 minutes, ensuring the data lake is continuously deduplicated.
* **The Anti-Pattern (Random UUIDs & Massive Scans):** Partitioning by event hour and product is correct, but sorting by a randomly generated `event_id` (like UUIDv4) causes min/max ranges to overlap across every single Parquet file. Spark's 15-minute `MERGE` is forced to open almost all files, blowing past the SLA and skyrocketing compute costs.
* **The Staff-Level Fix (Time-Sorted IDs & Bloom Filters):**
  1. **Bloom Filters:** Enable file-level Bloom filters in Iceberg to allow fast point-lookups for high-cardinality keys.
  2. **Upstream Contracts (Time-Sorted IDs):** Enforce an architectural guideline where product teams use chronologically sortable IDs (e.g., ULID, UUIDv7, Twitter Snowflake). The timestamp prefix physically groups the data, creating tightly bounded min/max statistics and allowing Spark to instantly skip 90% of files during the `MERGE`.

## 4. The SLA Distinction: Query Latency vs. Data Freshness
* **The Trap:** Confusing the time it takes to *query* data with the time it takes for data to *arrive*.
* **Sub-Second Query Latency (Read SLA):** This is achieved by the serving layer (ClickHouse). When a PM opens a dashboard, the query executes against ClickHouse's highly indexed, local `MergeTree` storage, rendering the dashboard in under a second.
* **15-30 Minute Data Freshness (End-to-End SLA):** This is the ingestion pipeline latency. It takes 15 minutes for Spark to micro-batch data into Iceberg, and another 15-30 minutes for Airflow to pull that incremental data into ClickHouse.
* **The Staff-Level Takeaway:** The dashboard loads instantly (sub-second query), but the data it displays represents the state of the world up to 30-45 minutes ago. Deliberately decoupling these two SLAs allows you to choose the right tool for each constraint (ClickHouse for fast reads, Spark/Iceberg for resilient batched writes) without over-engineering a true millisecond-streaming pipeline (like Flink), which would be massively expensive and unnecessary for this product use case.

## 5. The OLAP Serving Layer & Synchronization
* **The Goal:** Sub-second query latency on dashboards for hundreds of concurrent Product Managers over 50 TB/day of data.
* **The Anti-Pattern (Federated Queries):** Running federated queries directly from ClickHouse against raw Parquet files in Iceberg (object storage). I/O latency makes sub-second SLAs impossible at scale.
* **The Staff-Level Fix (Native Ingestion via Orchestrator):** 1. Physically ingest data into ClickHouse's native highly-indexed columnar storage (`MergeTree` engine family).
  2. Use a managed orchestrator (e.g., Apache Airflow MWAA) to run a 30-minute synchronization loop.
  3. Airflow polls Iceberg's metadata to discover the exact snapshot diff, then triggers a batch pull to copy *only* the incremental data to ClickHouse.
* **State Management:** Keep Airflow DAGs stateless by storing the "high-water mark" (last successfully processed Iceberg snapshot ID) in a durable external database like DynamoDB or RDS.

## 6. Distributed Atomicity & Airflow Retries
* **The Edge Case:** Airflow successfully writes a 2 TB batch into ClickHouse, but a network crash occurs before it can update the high-water mark in DynamoDB. Upon retry, Airflow will attempt to insert the same 2 TB batch.
* **The Anti-Pattern (`ReplacingMergeTree`):** Using ClickHouse's `ReplacingMergeTree` to handle the duplicates. This accepts the duplicate rows and merges them in the background. To get accurate dashboard metrics immediately, queries must use the `FINAL` keyword, forcing massive read-time compute and destroying the sub-second latency SLA.
* **The Staff-Level Fix (Insert Deduplication Tokens):** Utilize ClickHouse's native batch-level idempotency by passing the Iceberg Snapshot ID as the `insert_deduplication_token`. If Airflow retries the same task, ClickHouse recognizes the token and instantly skips the duplicate batch at the front door with almost zero compute overhead.

## 7. Resilience & Exactly-Once Semantics (AZ Failures)
* **The Threat:** An Availability Zone crashes precisely after Spark writes the Parquet files and commits the Iceberg snapshot, but *before* it can write the "Batch Complete" status to its S3 Checkpoint WAL. The restarted Spark cluster reads the old checkpoint and re-pulls the exact same Kafka offsets.
* **The Staff-Level Fix (Idempotent Sinks):** Avoid building custom, fragile state-tracking logic. Because the Spark write operation is physically designed as a `MERGE` keyed on the time-sorted `event_id`, the write is inherently idempotent. When Spark blindly reruns the batch, the execution engine evaluates the records against the existing Iceberg state and performs a no-op for the duplicates, mathematically guaranteeing exactly-once downstream delivery.