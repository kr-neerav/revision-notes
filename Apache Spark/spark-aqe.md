# Spark Adaptive Query Execution (AQE) – Revision Notes

## 1. The Static Planning Problem (Why AQE Exists)
* **The Flaw:** Standard Catalyst is a *compile-time* optimizer. It generates physical plans based on initial table statistics.
* **The Reality:** Transformations (like `filter`) drastically change data size mid-flight.
* **The Result:** "Blind execution." Spark executes heavy operations (like Shuffles and Sort-Merge Joins) that were planned at compile time, even if the runtime data volume has shrunk to a few kilobytes.

## 2. The AQE Framework (How It Works)
AQE allows Spark to pause, measure real data, and change the physical plan mid-flight.
* **Query Stages:** Spark execution is broken into stages. The boundary is always an Exchange (Shuffle/Broadcast).
* **Materialization Points:** Executors write intermediate shuffle files to local disk.
* **The AQE Loop:** 1. Pause at the stage boundary.
  2. Measure the actual size/rows of the materialized shuffle files.
  3. Feed metrics back to Catalyst to re-optimize remaining stages.
* *Note: If a pipeline has no shuffles (only narrow transformations like map/filter), AQE cannot intervene.*

## 3. Dynamically Coalescing Shuffle Partitions
Solves the `spark.sql.shuffle.partitions = 200` "Goldilocks" problem (too many partitions = scheduling overhead; too few = OOMs).
* **Mechanism:** Over-partition first, merge later.
* **Configuration:** Set `spark.sql.adaptive.coalescePartitions.initialPartitionNum` to a massive number (e.g., `8192`) to handle peak volume safely.
* **Execution:** At the materialization point, AQE measures the tiny shuffle files and dynamically *coalesces* (merges) them into optimally sized chunks (target `64MB` to `128MB`).
* *Rule:* AQE can merge partitions, but it cannot split them (except during skew handling).

## 4. Dynamically Switching Join Strategies
Prevents massive Sort-Merge Joins when a filtered dataset becomes tiny.
* **Mechanism:** If AQE measures that one side of a join has shrunk below `spark.sql.autoBroadcastJoinThreshold` (default `10MB`), it throws away the Sort-Merge Join.
* **Execution:** The Driver collects the tiny dataset, broadcasts it to all executors, and executes a lightning-fast **Broadcast Hash Join**.
* *Rule:* AQE can only *downgrade* to a cheaper join strategy (Sort-Merge -> Broadcast), never upgrade.

## 5. Dynamically Optimizing Skew Joins
Automates the mitigation of data skew without requiring manual code "salting."
* **Detection:** AQE flags a partition as skewed if it meets two criteria:
  1. Size > `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` (Default: `256MB`)
  2. Size > `spark.sql.adaptive.skewJoin.skewedPartitionFactor` * median partition size (Default: `5x`)
* **Mitigation:** 1. **Split:** Breaks the massive skewed partition into smaller sub-partitions.
  2. **Duplicate:** Copies the corresponding partition on the other side of the join to match the new sub-partitions.
  3. Parallelizes the hotspot across multiple executors seamlessly.

## 6. Production Tradeoffs & Edge Cases
AQE introduces **latency overhead** because the pause-measure-plan loop takes wall-clock time.

| Scenario | Impact | Action |
| :--- | :--- | :--- |
| **Low-Latency / Sub-second Queries** | Pause-and-plan overhead destroys SLAs. | Disable AQE (`enabled = false`). |
| **Perfectly Static Batch Jobs** | Adds overhead with no optimization gain. | Disable AQE (or leave on as insurance). |
| **Volatile / Unpredictable ETL** | Saves jobs from OOMs and massive delays. | Enable AQE globally. |

### Where AQE Fails (Requires Manual Salting/Tuning)
1. **Structured Streaming:** Cannot easily pause stateful micro-batches.
2. **Double Skew:** If both sides of the join are massive and skewed on the same key, AQE duplicating one side will cause an OOM.
3. **RDD API:** AQE only works on DataFrames/Spark SQL.
4. **"Small" Skew + Heavy CPU:** If a skewed partition is <256MB but runs an expensive UDF, AQE ignores it.