# Architecture Revision Notes: Java Garbage Collection

## Overview
The Java Garbage Collector (GC) is the background subsystem responsible for automating memory management in the JVM. Its primary goal is to maintain the illusion of infinite memory by reclaiming unreachable objects, shifting the performance bottleneck from manual memory allocation to CPU cycles and application latency (Stop-The-World pauses).

---

## 1. Reachability and GC Roots
The JVM does not track dead objects; it tracks live ones using a tracing mechanism to determine what is safe to delete.

* **The Reachability Rule:** Any object that can be traced back to a starting point (a GC Root) is considered "alive." Everything the GC cannot reach is implicitly garbage.
* **GC Roots:** These are active anchors in your application, primarily local variables on active thread stacks and static class variables.
* **Memory Leaks:** Unintentionally holding a strong reference to an object from a GC Root (e.g., a static `HashMap` acting as an unbounded cache) prevents the GC from reclaiming it, eventually causing a fatal `OutOfMemoryError`.

---

## 2. The Generational Hypothesis
JVM engineers optimize garbage collection sweeps based on the universal empirical observation that "most objects die young."

* **Young Generation:** Composed of Eden and Survivor spaces. New objects are born here. They are cleaned frequently and blazingly fast via Minor GCs.
* **Old Generation (Tenured):** Long-lived objects (like configuration and connection pools) that survive enough Minor GCs are promoted here. This space is cleaned rarely via computationally expensive Major GCs.
* **Premature Promotion:** If transient objects are too large or the Young generation is too small, short-lived garbage spills directly into the Old Generation, causing catastrophic Full GCs.

---

## 3. Mark, Sweep, and Compact
The foundational algorithm for physically reclaiming memory and managing the heap space.

* **Mark and Sweep:** The GC traverses the object graph to mark live objects, then sweeps un-marked objects to update the free memory list.
* **Memory Fragmentation:** Sweeping leaves gaps. If your application needs contiguous space for a large object and cannot find it, the JVM panics, even if total free memory is high.
* **Compaction and STW:** To fix fragmentation, the GC slides surviving objects together. Because objects are physically moving, the JVM must trigger a Stop-The-World (STW) pause, freezing all application threads to update memory pointers safely.

---

## 4. G1GC (Garbage-First)
The modern default collector that replaces massive contiguous continents with smaller, flexible memory regions to bound STW pause times.

* **Region-Based Heap:** The heap is chopped into thousands of chunks (1MB-32MB). Any region can dynamically act as Eden, Survivor, or Old generation space.
* **Liveness Accounting:** Background threads concurrently trace the heap while the application runs to build a "scoreboard" of which regions contain the most garbage.
* **Soft Pause Targets:** G1GC evacuates only the most profitable regions within a user-defined time budget (e.g., `-XX:MaxGCPauseMillis=200`), avoiding massive Full GCs under normal operating conditions.

---

## 5. Ultra-Low Latency (ZGC & Shenandoah)
Collectors designed for strict SLAs, guaranteeing sub-millisecond pause times regardless of whether the heap is 10 Megabytes or 16 Terabytes.

* **Concurrent Compaction:** The holy grail of GC. These collectors move objects physically to defragment the heap while application threads are actively reading and writing to them.
* **Load Barriers & Colored Pointers:** Every memory read is intercepted by a tiny assembly snippet. If an application thread tries to read an object that is currently moving, the barrier transparently "heals" the stale pointer and routes the thread to the new address.
* **The Latency Tradeoff:** You pay a constant CPU tax on every single memory read, lowering maximum overall throughput in exchange for eliminating latency spikes.

---

## 6. Production Realities & Debugging
Misconfiguring the JVM or abusing the heap introduces severe operational risks in distributed systems.

| Failure Mode | Root Cause | Diagnostic & Fix |
| :--- | :--- | :--- |
| **Latency Spikes (Expansion)** | `-Xms` (initial) and `-Xmx` (max) heap sizes are different, forcing STW pauses to negotiate memory with the OS under peak load. | **Lock the heap:** Always set `-Xms` equal to `-Xmx` on startup. |
| **OOMKilled by OS (Exit Code 137)** | Total JVM memory (Heap + Off-Heap) exceeded the container's hard limit (e.g., Kubernetes cgroups). The JVM is killed silently without throwing a Java error. | **Mind the Container Limit:** Always leave a buffer between `-Xmx` and the container's absolute memory limit. |
| **The Staircase of Death** | A slow memory leak where each Full GC reclaims slightly less memory than the last. | Enable `-Xlog:gc*`. Trigger a Heap Dump (`.hprof`) and use Eclipse MAT's **Dominator Tree** to find the GC Root. |
| **Spark Executor Timeout** | Extreme object allocation (e.g., massive joins) exhausted G1GC's regions, forcing it to abandon its target and fall back to a brute-force Full GC. | G1GC's pause targets are *soft*. For strict SLAs, use ZGC. For pure batch throughput, switch to Parallel GC. |