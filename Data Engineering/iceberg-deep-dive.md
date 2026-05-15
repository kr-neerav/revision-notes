# Expanded Staff-Level Architecture: High-Throughput Iceberg Telemetry Pipeline
*Comprehensive Revision Notes: Designing for Strict SLAs, Concurrency, and Scale*

## 1. The Core Challenge & The Optimistic Concurrency Control (OCC) Trap
* **The Goal:** Ingest high-volume telemetry data (billions of events) with a strict **5-minute freshness SLA**.
* **The Constraint:** Network issues (e.g., flaky VPNs, offline mobile clients) cause data to arrive up to 30 minutes late, creating out-of-order event streams.
* **The Trap (Why standard `MERGE` fails):** Iceberg uses Optimistic Concurrency Control (OCC). 
  * If a Spark Streaming job tries to run an inline `MERGE INTO` (to handle duplicates as they arrive) at the exact same time a background compaction job (Binpack/Sort) is rewriting the underlying small Parquet files, a conflict occurs. 
  * Because the compaction job changed the data files that the streaming job was trying to update, the streaming job suffers a **Hard Fail** (rebasing failure) and is forced to retry. 
  * Frequent retries under high load will completely destroy your 5-minute SLA.

## 2. The "Side-Load" Pattern (Write Isolation)
To guarantee the ingestion SLA, we must completely eliminate write conflicts by physically isolating the "Hot Path" from the "Chaos Path".
* **Hot Path (Append-Only):** On-time data is streamed directly to the Main Iceberg Table using pure `APPEND` operations. `APPEND` operations rarely conflict because they only add new files to the manifest, rather than modifying existing ones.
* **Chaos Path (Staging):** Late-arriving data (`event_time < now - 30m`) is routed to a separate Iceberg **Staging Table**. This isolates the messy, out-of-order data from the high-speed main pipeline.

## 3. Two-Layer Deduplication (Balancing Read vs. Write Compute)
Because we are using `APPEND` only on the Hot Path, duplicates will slip into the tables. We shield downstream consumers using a two-layer approach:
* **Logical Layer (Real-Time Reads):** We expose a unified SQL View to consumers. 
  * *Query:* `SELECT * FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY processing_time DESC) as rn FROM (SELECT * FROM main UNION ALL SELECT * FROM staging)) WHERE rn = 1;`
  * *Why:* This deduplicates data instantly at query time, guaranteeing accuracy for real-time dashboards.
* **Physical Layer (Batch Cleanup):** An hourly Spark job merges the Staging table into the Main table.
  * *Why:* If we rely only on the Logical View, the queries will eventually become too slow and expensive. The physical `MERGE` resolves duplicates on disk and keeps the Staging table small, ensuring the logical view remains highly performant.

## 4. Idempotency & State Management (The Cleanup Handshake)
We must ensure the hourly batch job doesn't duplicate data or fail catastrophically if it crashes mid-run.
* **The Anti-Pattern (The Moving Target):** Do not use `SELECT * FROM staging` followed by a `DELETE`. 
  * Running a physical `DELETE` on a Data Lake requires rewriting files (Copy-on-Write) or managing complex delete files (Merge-on-Read). It is computationally expensive, risks partial failures, and creates a "moving target" if new data lands while the delete is running.
* **The Staff-Level Fix (Incremental Scans + DynamoDB):** 
  * We use an external state store (DynamoDB) to durably track the `last_processed_snapshot_id`.
* **The Execution Flow:**
  1. Job wakes up and reads the last `snapshot_id` from DynamoDB (e.g., Snapshot A).
  2. Spark queries Iceberg for *only the files added* between Snapshot A and the current state (Snapshot B).
  3. Spark executes an idempotent `MERGE INTO` the Main table using that delta.
  4. If successful, it updates DynamoDB to track Snapshot B.
  * *Resilience:* If the job crashes at Step 3, DynamoDB still points to Snapshot A. The next run retries the exact same delta. Zero duplicated effort, zero data loss.

## 5. Storage Maintenance & Self-Healing Pipelines
Since we abandoned manual `DELETE` statements in our pipeline, the Staging table will grow infinitely. We decouple maintenance from our critical path.
* **The Maintenance Fix (TTL):** We configure an Iceberg table property (e.g., `history.expire.max-snapshot-age-ms`) to 24 hours. A background AWS Glue or cron job runs `expireSnapshots` and deletes the physical, unreferenced S3 Parquet files automatically.
* **The Outage Edge Case:** If the pipeline goes down for 3 days, the 24-hour TTL job will physically delete the snapshot tracked in DynamoDB. When the pipeline restarts, the incremental scan will throw an `IllegalArgumentException` because its starting point is gone.
* **Self-Healing Logic:** 
  * Wrap the Spark read in a `try/catch` block.
  * If the specific snapshot exception is caught, dynamically query Iceberg's metadata table: `SELECT snapshot_id FROM staging.refs.snapshots ORDER BY committed_at ASC LIMIT 1`.
  * Update the Spark job to use this new, oldest available snapshot.
  * Automatically page the on-call engineer to trigger a backfill script for the data window that was permanently lost to the TTL.

## 6. S3 Throttling & Read/Write Isolation
* **The Threat:** A new ML cluster spins up 1,000+ concurrent workers to query the main table. This causes severe S3 API throttling (`503 Slow Down`) on your bucket, which cascadingly crashes your critical 5-minute ingestion stream because S3 drops its write requests.
* **Solution 1 (Storage Physics & Entropy):** Enable Iceberg's **Object Store Layout** (`write.object-storage.enabled = true`). 
  * *Why:* S3 scales its I/O limits based on prefixes. Standard layouts (`/table/data/partition/`) put all traffic on one prefix. Object Store Layout injects a deterministic hash (`/<hash>/data/...`), natively sharding the S3 I/O limits across hundreds of internal partitions from day one.
* **Solution 2 (Metadata Caching):** Introduce a caching layer (e.g., Alluxio) specifically for the ML workers. 
  * *Trade-off:* We accept *eventual consistency* (ML models might read a snapshot that is 10 minutes old) to strictly protect the *availability* of our write SLA. 
  * *Observability:* Deploy a lightweight canary job that pings the cache's current `snapshot_id` against S3's actual `snapshot_id`. If the cache falls too far behind, fire an alert to prevent ML teams from training on severely stale data.