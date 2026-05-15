# Staff-Level Architecture Notes: Unified Telemetry & State Ingestion Pipeline
*Comprehensive Revision Notes: Ingestion Mechanics, Enrichment Throttling, Storage Physics, and Compliance Lifecycle*

## 1. Source Mechanics & The CDC / Diff Trap
* **The Goal:** Ingest and unify high-volume telemetry data (Terabytes/day, 15-minute micro-batches) with application database state (Gigabytes/day, hourly syncs).
* **The Constraint:** We need to accurately track row deletions in the application database to keep analytics accurate, but we want to avoid heavy infrastructure.
* **The Anti-Pattern (Massive Diffs & Strict CDC):** Running a daily Full Outer Join on a 1TB database snapshot just to find a few megabytes of hard deletes wastes massive network/compute resources and introduces severe reliability risks (e.g., Spark OOM errors). Deploying strict Change Data Capture (CDC) via Debezium/DMS introduces a high "infrastructure tax" that is overkill for this specific use case.
* **The Staff-Level Fix (Contracts & Tombstones):** Push the data contract upstream. The application team implements a **Soft Delete (tombstone)** pattern (e.g., flipping an `is_deleted` boolean) instead of hard deleting rows. The application team exports an hourly snapshot to S3 directly from a **Read Replica** to completely isolate analytical reads from operational database performance.

## 2. API Enrichment & The Thundering Herd
* **The Threat:** We must enrich incoming telemetry via an external API. A synchronous REST API call for every single event across terabytes of data will cause a "Thundering Herd" cache stampede, DDoS-ing the external API and crashing our pipeline.
* **The Solution (Intra-batch Deduplication):** We rely on local executor caching and micro-batch awareness.
* **The Execution Flow:**
  1. Within a 15-minute Spark micro-batch, an executor identifies all *unique* missing keys (e.g., `user_id`) in its local chunk.
  2. The executor makes a single, bulk API call for those unique keys.
  3. The results are cached locally on the executor to enrich the rest of the batch.
  * *Trade-off:* We take on the complexity of stateful batching at the ingestion layer to guarantee data completeness downstream while strictly protecting upstream API rate limits.

## 3. Storage Architecture: The Immutable Landing Zone
* **The Core Principle:** "Save-to-disk-first".
* **Execution:** All incoming data (15-minute telemetry micro-batches and hourly database S3 exports) lands immediately in an S3 Raw layer.
* **Why:** We perform minimal to no transformations at this stage. This ensures a perfectly auditable history of the data exactly as it was received, allowing for safe pipeline replays and backfills during downstream production outages.

## 4. Orchestration & The "Waiting for Godot" Problem
* **The Constraint:** Downstream "Clean/Join" jobs must only trigger when an entire hour of data from multiple distinct cadences is completely available.
* **The Anti-Pattern (Fragile Signals):** Schedule-based triggers fail if upstream data is delayed. Querying massive S3 buckets for "file-exists" checks is expensive and prone to race conditions.
* **The Staff-Level Fix (Iceberg State Machine):** Decouple compute from orchestration using a dedicated, tiny Apache Iceberg table (`telemetry_ingestion_state`) as an active state machine.
* **The Execution Flow:**
  1. **Active Participant:** Spark processes a 15-minute micro-batch, commits to the main `telemetry_raw` table, and appends a row to the state table indicating that window is complete (e.g., `window = 1, status = 'COMMITTED'`).
  2. **Passive Observer:** The orchestrator (Airflow/Dagster) runs a lightweight SQL sensor against the state table: `SELECT COUNT(DISTINCT micro_batch_window) FROM telemetry_ingestion_state WHERE target_hour = 'X' AND status = 'COMMITTED'`.
  3. If count equals 4, it triggers the downstream job; otherwise, it sleeps.

## 5. Late Arrivals & System Resilience
* **The Edge Case:** If an upstream telemetry API goes down, micro-batch #3 is missing, causing the SQL sensor to stall indefinitely.
* **The Maintenance Fix (Grace Periods & Reconciliation):** The orchestration logic implements a strict timeout (grace period). If the 4th micro-batch does not arrive within the SLA, the sensor unblocks and pushes available data forward. We prioritize *Data Freshness* over absolute *Data Completeness* for the real-time SLA, relying on a separate asynchronous reconciliation job to sweep up late-arriving data.

## 6. Storage Physics & The Two-Tier Compaction Strategy
* **The Threat:** Landing terabytes of data via 15-minute micro-batches directly into hourly partitions creates thousands of tiny Parquet files. This severely degrades downstream I/O performance and bloats Iceberg metadata. Furthermore, concurrent compaction and streaming writes cause Optimistic Concurrency Control (OCC) conflicts, crashing the pipeline.
* **The Staff-Level Fix (Decoupled Workflows):** We physically isolate read/write paths and use two distinct compaction workflows, as one job cannot efficiently handle both recent "Small Files" and historical "Tombstones."
  * **The Hot Path (Write):** Spark strictly `APPENDS` data to the latest partitions.
  * **Workflow A - The "Sweeper" (Small File Compaction):** Runs frequently (e.g., every 4-12 hours). Targets only the recent 48 hours to stitch tiny 15-minute Parquet files into optimized 512MB files for fast downstream querying (`where event_hour >= NOW() - INTERVAL 48 HOURS`).

## 7. Schema Evolution & Data Contracts
* **The Threat:** Upstream teams dynamically change JSON payloads. Spark inferring schemas on the fly wastes compute; hardcoded schemas cause pipeline drops or silent failures.
* **The Staff-Level Fix (Schema Registry):**
  * Move from implicit trust to explicit **Data Contracts** ("Schema on the Street").
  * All incoming records are tagged with a schema version managed by a centralized Schema Registry.
  * Spark retrieves the exact `StructType` from the registry, bypassing inference costs entirely. Payloads violating the contract are safely routed to a Dead Letter Queue (DLQ).

## 8. Data Retention & Time Travel 
* **The Constraint:** Infinite Iceberg snapshots for a 15-minute ingestion pipeline cause metadata bloat and OOM errors during query planning. Infinite physical files cause massive cloud storage bills.
* **The Strategy (Decoupling Logical vs. Physical Retention):**
  * **Metadata (Iceberg Time Travel):** Capped at **3 to 7 days**. A daily job runs `expire_snapshots` to keep manifest trees lean.
  * **Physical Storage (S3):** Capped at **7 to 14 days** in standard S3 (Hot) for late-arriving reconciliation and incident debugging. S3 Lifecycle rules automatically transition raw immutable data to Glacier Deep Archive (Cold) for long-term compliance storage.

## 9. Compliance: GDPR & "Right to be Forgotten"
* **The Challenge:** Physically scrubbing a single user's PII from a massive, immutable Append-Only Raw layer without rewriting years of historical data or disrupting the ingestion pipeline.
* **The Staff-Level Fix (Iceberg Merge-on-Read / Tombstones):**
  1. **Logical Delete (Immediate):** Execute a standard `DELETE` query. Iceberg writes a positional/equality Delete File (tombstone). This satisfies the immediate compliance SLA by hiding the user from all downstream queries.
  2. **Workflow B - The "Deep Clean" (GDPR Compaction):** Batched weekly. Unlike the Sweeper, this spans years of historical data but uses surgical precision. We pass the `delete-file-threshold = 1` option. Iceberg's engine reads its own manifests, skips 99% of clean historical files, and *only* spends compute rewriting the specific Parquet files that are afflicted by tombstones.
  3. **The True Purge:** The original Parquet files containing the PII are permanently, physically deleted from the S3 bucket when the standard `expire_snapshots` job drops the older metadata references (Step 8).