---
tags:
  - performance
  - concurrency
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Cache Coherency

Cache coherency — specifically the MESI protocol — is the dominant cost in multi-threaded systems at scale, and the fundamental hardware problem that [[thread-per-core]] architecture solves.

## The cost

When a work-stealing scheduler (like [[tokio]]'s) migrates a task from core A to core B, every cache line that task touches must be invalidated on A and reloaded on B — **50–200 nanoseconds per cache line** under contention. [[false-sharing|False sharing]], where independent data happens to occupy the same cache line, can degrade throughput by **10x or more**. Even uncontended atomic operations (the backbone of lock-free work-stealing queues) carry a **5x penalty** over non-atomic equivalents due to LOCK prefix instructions forcing memory ordering.

The [[faa-vs-cas|FAA-vs-CAS]] gap (40 ns vs 75 ns at 144 threads) is entirely a cache coherence story: CAS retries bounce the contended line repeatedly between cores, while FAA completes in a single coherence transaction.

## Why thread-per-core works

By pinning one thread per physical core with its own allocator, network queue, and storage partition, [[thread-per-core]] eliminates all inter-core cache coherency traffic. [[seastar|ScyllaDB's Seastar]] proved this enables near-linear scaling. The Rust [[thread-per-core]] runtimes — [[monoio]], [[compio]], [[glommio]] — achieve 2–3x [[tokio]] throughput at 16+ cores primarily by avoiding these costs.

## Cache-friendly data structure design

Cache coherency costs are one dimension; cache *utilization* is the other. The [[soa-vs-aos]] pattern (1.4–12.7x speedup), [[b-tree]] over red-black trees (3–18x), and [[csr-graph|CSR]] over adjacency lists (40–250x) all demonstrate that contiguous memory layouts exploiting cache lines dominate pointer-chasing alternatives. The [[lmax-disruptor]] explicitly pads sequence counters to cache-line boundaries to prevent false sharing — and the modern correction to that practice is [[false-sharing|128-byte padding rather than 64-byte]], because x86's adjacent cache-line prefetcher pulls in the neighbor on every fetch. The 64→128 fix alone is worth ~1.7× on contended ring buffers. See [[fastest-data-structures]] for the full benchmarked landscape.
