# Architecture Revision Notes: Apache Spark Project Tungsten

## Overview
Project Tungsten is the fundamental architectural rewrite of Apache Spark's execution engine. Its primary goal is to shift the performance bottleneck from disk/network I/O to CPU and memory bandwidth by bypassing JVM limitations and exploiting modern hardware (L1/L2 caches, SIMD registers).

---

## 1. The JVM Bottleneck
Standard Java objects are structurally hostile to high-throughput data processing.

* **Object Bloat:** A standard JVM object has massive overhead (Mark Word, Klass Pointer, 8-byte alignment padding). Raw data is often inflated by 4x to 8x (e.g., a 6-byte string takes ~48 bytes).
* **GC Death Spiral:** Creating billions of transient objects for intermediate pipeline operations fills the Eden space, causing cascading Minor GCs and Stop-The-World (STW) Major GCs, leading to distributed task timeouts.
* **Serialization Tax:** Moving objects across the network requires expensive CPU cycles to traverse object graphs and serialize metadata.

---

## 2. Explicit Memory Management & UnsafeRow
Tungsten abandons JVM objects for a custom binary format (`UnsafeRow`) and C-style memory management.

* **UnsafeRow Layout:** A flat, contiguous byte array with three strict regions:
  1. **Null BitSet:** 1 bit per field to track nulls.
  2. **Fixed-Length Data:** 8-byte slots. Stores primitives directly, or stores a `[32-bit Offset | 32-bit Length]` pointer for variable-length data.
  3. **Variable-Length Data:** The actual bytes for strings/arrays, appended at the end.
* **Bypassing the GC:** Spark allocates massive contiguous memory pages (often off-heap via OS `malloc`) using the internal `sun.misc.Unsafe` API. The GC sees one giant array instead of millions of objects, dropping GC pauses to near zero.
* **Zero-Copy I/O:** Raw byte arrays can be streamed directly to network sockets during shuffles without serialization overhead.

---

## 3. Cache-Aware Computation
Modern CPUs are bottlenecked by main memory latency (~100ns) compared to L1 cache access (~1ns). 

* **The 64-Byte Rule:** CPUs fetch data in 64-byte chunks (Cache Lines). 
* **Pointer Chasing:** JVM objects scatter data across the heap. Evaluating filters causes constant cache misses and CPU stalls.
* **Hardware Prefetching:** Because `UnsafeRow` packs data sequentially, the CPU's hardware prefetcher detects the sequential read pattern and pre-loads the next rows into the L1 cache before the software asks for them, effectively masking RAM latency.

---

## 4. Whole-Stage Code Generation (WSCG)
Tungsten replaces the traditional Volcano Iterator Model with dynamic runtime compilation.

* **The Volcano Problem:** The `Scan -> Filter -> Project` model requires calling `next()` on every row for every operator. This creates billions of virtual function dispatches, destroying CPU pipelining and branch prediction.
* **The Tungsten Solution:** Spark acts as a compiler. It takes the entire physical query plan and writes a single, highly optimized Java `for` loop at runtime (compiled via Janino).
* **Operator Fusion:** Multiple pipeline steps are fused into one loop. Data stays in the CPU registers between operations instead of being written back to memory.

---

## 5. Vectorized Processing & SIMD
While WSCG is incredibly fast, it is still *Scalar* (processing one row at a time). 

* **SIMD (Single Instruction, Multiple Data):** Modern CPUs have ultra-wide registers (e.g., AVX-512) that can apply a single math operation to multiple data points simultaneously (e.g., adding 16 integers in one clock cycle).
* **ColumnarBatch:** SIMD requires perfectly contiguous, typed data (like `int[]`). `UnsafeRow` (which mixes types) defeats SIMD. To solve this, Spark reads columnar formats (like Parquet) into memory as a `ColumnarBatch` (a collection of primitive arrays) to execute vectorized operations before pivoting back to `UnsafeRow` for shuffles.

---

## 6. Production Realities & Debugging
Bypassing the managed runtime (JVM) introduces C++ style operational risks.

| Failure Mode | Standard Spark | Project Tungsten |
| :--- | :--- | :--- |
| **Out of Memory** | `java.lang.OutOfMemoryError` in logs. | Off-heap memory limits trigger Linux OOM Killer. Executor dies silently with **Exit Code 137**. |
| **Exception Traces** | Stack trace points to a logical operator (e.g., `Filter`). | Operators are fused into `GeneratedClass`. Requires using `df.explain("codegen")` to read raw Java and map line numbers. |
| **Network Routing** | N/A | Must pivot `ColumnarBatch` back to `UnsafeRow` for shuffles. Columnar data is strictly for compute; Row data is strictly for network routing. |