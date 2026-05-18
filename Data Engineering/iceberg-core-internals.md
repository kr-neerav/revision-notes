# Staff-Level Architecture Notes: Apache Iceberg Core Internals
*Comprehensive Revision Notes: Metadata Tracking, Optimistic Concurrency, and O(1) Query Pruning*

## 1. The Source of Truth: Immutability & State
* **The Core Principle:** Iceberg never modifies data or metadata in place. State is managed by atomic pointer swaps to an immutable history of the table.
* **The Absolute Source of Truth (`TableMetadata`):** A JSON file that holds the schema history, partition specs, and the critical `current-snapshot-id`. When a query runs, the catalog returns the path to the current `metadata.json` file.
* **The Point-in-Time State (`Snapshot`):** A snapshot is *not* a standalone file in S3. It is a tiny JSON object embedded directly inside the `metadata.json` file's `snapshots` array. It contains metrics and the vital pointer to the `Manifest List`.
* **The Mechanism (Time Travel):** Time travel is not a complex compute operation; it simply involves reading a historical `Snapshot` JSON object from the `TableMetadata` array instead of the `current-snapshot-id`.

## 2. The O(1) Query Pruning Engine (Manifest Tracking)
* **The Goal:** Plan queries over petabytes of data without executing heavy, latency-inducing `ls` (list) operations on cloud object storage.
* **The Hierarchy:** `Snapshot (JSON)` → `Manifest List (Avro)` → `Manifest File (Avro)` → `Data File (Parquet)`.
* **The Manifest List:** An Avro file that acts as an index. It stores the `lower_bound` and `upper_bound` for partition fields. This allows the query engine (e.g., the Spark Driver) to instantly drop entire partitions from the query plan without downloading the lower-level manifests.
* **The Manifest File:** An Avro file that stores column-level Min/Max stats (`lower_bounds`, `upper_bounds`) and `null_value_counts` for *every individual Parquet file*. This allows the engine to drop 99% of files before assigning read tasks to workers.

## 3. The N+1 Problem & Parquet Footers
* **The Trap:** Parquet natively stores Min/Max stats in its file footer. However, relying on the footer requires the query engine to make 1,000,000 S3 `GET` requests to open 1,000,000 files *just to read their stats*. This is what made legacy Hive metastores fail at scale.
* **The Staff-Level Fix (Centralized Stats):** Iceberg extracts these stats at write-time and hoists them into the row-based Avro `Manifest File`. 
* **The Result:** The query engine makes a single S3 request to download an 8MB Manifest File and instantly evaluates the exact footers of 10,000 Parquet files simultaneously without opening the actual data.

## 4. Concurrent Writes: Optimistic Concurrency Control (OCC)
* **The Challenge:** Supporting concurrent writes from thousands of Spark workers without expensive distributed locks or single points of failure.
* **The Staff-Level Fix (OCC & Atomic Swaps):** 
  1. The engine does the heavy lifting (writing Parquet files to storage) assuming it will succeed.
  2. `SnapshotProducer` builds a new metadata state and attempts an atomic Compare-and-Swap (CAS) in the Catalog.
  3. If another writer moved the pointer first, Iceberg catches a `CommitFailedException`.
* **The Retry Magic:** The engine does *not* rewrite the heavy Parquet files. It pulls the fresh metadata, validates that its operation (e.g., `APPEND`) doesn't logically conflict with the new state, rebuilds the tiny manifest list, and retries the CAS swap using exponential backoff.

## 5. Schema Evolution via Column IDs
* **The Challenge:** Safely dropping, renaming, or re-adding columns over years of historical data.
* **The Trap:** Relying on column names or file positioning corrupts historical reads when a column is dropped and a new one is added with the exact same name later.
* **The Staff-Level Fix (Integer Column IDs):** Iceberg assigns a unique integer ID to every column. `TableMetadata` keeps an infinite array of all historical schemas.
* **The Execution:** When reading a 2-year-old Parquet file, Iceberg uses the historical schemas as a translation dictionary, safely mapping the old file's column IDs to the modern query without data corruption. Because schemas are tiny JSON arrays, keeping them forever is virtually free.

## 6. S3 Throttling on High-Concurrency Reads
* **The Threat:** Thousands of concurrent query workers hitting the same `s3://bucket/table/metadata/` prefix causes S3 to throttle with `503 Slow Down` errors (AWS limits: 5,500 GET/s per prefix).
* **The Staff-Level Fix (Entropy Injection):** Enable **Object Store Layout** (`write.object-storage.enabled = true`). Iceberg injects a pseudo-random hash at the front of the S3 URI (`s3://bucket/table/<hash>/metadata/`). This naturally shards the manifests and data files across hundreds of distinct S3 prefixes, mathematically scaling AWS's I/O limits and providing virtually infinite read scalability.
* **Query Planning Distribution:** Furthermore, the `Manifest List` and `Manifest Files` are only read by the query *Coordinator/Driver*, not the thousands of worker nodes, ensuring metadata files only receive one GET request per query.
