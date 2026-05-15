# Staff-Level Architecture Notes: Unified Telemetry & State Ingestion Pipeline
*Comprehensive Revision Notes: Ingestion Mechanics, Enrichment Throttling, and Event-Driven Orchestration*

## 1. Source Mechanics & The CDC / Diff Trap
* **The Goal:** Ingest and unify high-volume telemetry data (Terabytes/day, 15-minute micro-batches) with application database state (Gigabytes/day, hourly syncs).
* **The Constraint:** We need to accurately track row deletions in the application database to keep analytics accurate, but we want to avoid heavy infrastructure.
* **The Anti-Pattern (Massive Diffs & Strict CDC):** * Running a daily Full Outer Join on a 1TB database snapshot just to find a few megabytes of hard deletes is a massive waste of network and compute resources, and introduces severe reliability risks (e.g., Spark OOM errors on large shuffles).
  * Deploying strict Change Data Capture (CDC) via Debezium/DMS introduces a high "infrastructure tax" and operational complexity that is overkill for this specific telemetry use case.
* **The Staff-Level Fix (Contracts & Tombstones):** * Push the data contract upstream. The application team implements a **Soft Delete (tombstone)** pattern (e.g., flipping an `is_deleted` boolean) instead of hard deleting rows.
  * The application team exports an hourly snapshot to S3 directly from a **Read Replica** to completely isolate analytical reads from operational database performance.

## 2. API Enrichment & The Thundering Herd
* **The Threat:** We must enrich incoming telemetry via an external API. If we attempt a synchronous REST API call for every single event across terabytes of data, we will cause a "Thundering Herd" cache stampede, effectively DDoS-ing the external API and crashing our ingestion pipeline.
* **The Solution (Intra-batch Deduplication):** We rely on local executor caching and micro-batch awareness rather than centralized caching (which adds network hops) or asynchronous downstream enrichment.
* **The Execution Flow:**
  1. Within a 15-minute Spark micro-batch, an executor identifies all *unique* missing keys (e.g., `user_id`) in its local chunk.
  2. The executor makes a single, bulk API call for those unique keys.
  3. The results are cached locally on the executor to enrich the rest of the batch.
  * *Trade-off:* We take on the complexity of stateful batching at the ingestion layer, but we guarantee data completeness downstream while strictly protecting the upstream API rate limits.

## 3. Storage Architecture: The Immutable Landing Zone
* **The Core Principle:** "Save-to-disk-first".
* **Execution:** All incoming data—both the 15-minute telemetry micro-batches and the hourly database S3 exports—lands immediately in an S3 Raw layer.
* **Why:** We perform minimal to no transformations at this stage. This ensures a perfectly auditable history of the data exactly as it was received, allowing for safe pipeline replays and backfills during downstream production outages.

## 4. Orchestration & The "Waiting for Godot" Problem
* **The Constraint:** We have multiple distinct cadences running (15-min telemetry, hourly DB syncs). A downstream "Clean/Join" job must only trigger when an entire hour of data is completely available.
* **The Anti-Pattern (Fragile Signals):** Relying on schedule-based triggers (e.g., running every hour on the hour) fails instantly if upstream data is delayed. Similarly, querying massive S3 buckets for "file-exists" checks to determine completeness is expensive and prone to race conditions.
* **The Staff-Level Fix (Iceberg State Machine):** We decouple the compute layer from the orchestration layer by using a dedicated, tiny Apache Iceberg table (`telemetry_ingestion_state`) as an active state machine.
* **The Execution Flow:**
  1. **Active Participant:** Spark processes a 15-minute micro-batch, commits the data to the main `telemetry_raw` table, and then appends a single row to the state table indicating that specific window (e.g., `window = 1, status = 'COMMITTED'`).
  2. **Passive Observer:** The orchestrator (e.g., Airflow/Dagster) runs a lightweight SQL sensor against the state table: `SELECT COUNT(DISTINCT micro_batch_window) FROM telemetry_ingestion_state WHERE target_hour = 'X' AND status = 'COMMITTED'`.
  3. If the count equals 4, the orchestrator triggers the downstream job. If not, it sleeps.

## 5. Late Arrivals & System Resilience
* **The Edge Case:** If an upstream telemetry API goes down and micro-batch #3 is missing, the SQL sensor will stall indefinitely, preventing the entire hour (and subsequent hours) from processing.
* **The Maintenance Fix (Grace Periods & Reconciliation):** * The orchestration logic implements a strict timeout (grace period). If the 4th micro-batch does not arrive within the acceptable SLA window, the sensor unblocks and pushes the available data forward.
  * We accept a trade-off prioritizing *Data Freshness* over absolute *Data Completeness* for the real-time SLA, relying on a separate, asynchronous reconciliation job to sweep up and process the late-arriving data later.