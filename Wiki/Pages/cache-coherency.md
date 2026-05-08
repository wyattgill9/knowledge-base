---
tags:
  - performance
  - concurrency
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Cache Coherency

Cache coherency is the protocol by which CPU cores keep each other informed about modifications to shared cache lines, and it is the dominant cost in multi-threaded systems at scale. The fundamental unit is a **64-byte cache line** bouncing between cores via the coherence protocol — and on modern hardware this transfer takes between **16 ns** (AMD Zen 3 same-CCX) and **150 ns** (Intel Sapphire Rapids cross-socket). Every [[inter-thread-communication|inter-thread mechanism]] ultimately bottlenecks on this physical floor; the engineering challenge is getting close to it without making the cost worse.

## MESI, MESIF, MOESI

| Vendor | Protocol | States |
|--------|----------|--------|
| Intel | MESIF | Modified, Exclusive, Shared, Invalid, Forward |
| AMD | MOESI | Modified, Owned, Exclusive, Shared, Invalid |
| ARM | MESI variants per implementation | Mostly MESI with vendor extensions |

Intel's MESIF adds a "Forward" state to designate one sharer as the canonical responder for read requests, reducing redundant snoop responses. AMD's MOESI adds an "Owned" state that lets a modified line be shared with another core *without* writing back to memory first — the dirty line stays in caches, with one core responsible for eventual writeback. This is part of why AMD Zen 3 has industry-leading core-to-core latency for shared modifications.

## Core-to-core latency by topology

| CPU | Same-cluster | Cross-cluster |
|-----|--------------|---------------|
| AMD Ryzen 9 5900X (Zen 3) | **16 ns** | 84 ns (cross-CCX via Infinity Fabric) |
| Intel i9-9900K (Coffee Lake) | **21 ns** | — (uniform 4-core ring bus) |
| Intel i9-12900K (Alder Lake P→P) | 35 ns | 50 ns (E→E) |
| Intel Xeon Max 9480 (Sapphire Rapids) | 61.5 ns | 150 ns (cross-socket) |
| Apple M1 Pro (P→P) | 40 ns | 145 ns (P→E cluster) |
| AWS Graviton 3 (64 cores) | 46 ns | — (uniform mesh) |

The table tells the story. **Topology dominates algorithm**: the same SPSC ring buffer hits 16 ns RTT between two Zen 3 cores in the same CCX and 107 ns between cores in different chiplets — same code, same data, **6.7× slower**. Hyperthreads on the same core can signal each other in **16.5 ns** since they share L1/L2, but hyperthread contention on execution units usually negates the proximity benefit. See [[core-pinning]] for the [[inter-thread-communication|HFT recipe]] that exploits topology deliberately.

## The cost in practice

When a work-stealing scheduler (like [[tokio]]'s) migrates a task from core A to core B, every cache line that task touches must be invalidated on A and reloaded on B — **50–200 nanoseconds per cache line** under contention. [[false-sharing|False sharing]], where independent data happens to occupy the same cache line, can degrade throughput by **10× or more**. Even uncontended atomic operations (the backbone of lock-free work-stealing queues) carry a **5× penalty** over non-atomic equivalents due to LOCK prefix instructions forcing memory ordering.

The [[faa-vs-cas|FAA-vs-CAS]] gap (40 ns vs 75 ns at 144 threads) is entirely a cache coherence story: CAS retries bounce the contended line repeatedly between cores, while FAA completes in a single coherence transaction. The Schweizer-Besta-Hoefler ETH Zurich study confirmed that CAS, FAA, and swap have *identical instruction-level latency* — the popular belief that "FAA is faster" comes from FAA's semantic guarantee against retries, not from cheaper instructions.

## CLDEMOTE and proactive cache management

Intel's `CLDEMOTE` instruction (introduced with Sapphire Rapids, 2024) hints to the hardware that a cache line should be pushed proactively toward L3, where another core can read it without an L1-to-L1 transfer. This breaks the worst-case "modified line in producer's L1, consumer reads, full coherence round-trip" pattern by pre-emptively demoting the line to a state where the consumer's read is cheap.

Intel's optimization study reported that for an 8+ thread inter-thread messaging workload, `CLDEMOTE` boosted aggregate core-to-core bandwidth from **7.3 GB/sec to 22 GB/sec** — a 3× improvement from a single instruction. The trade-off is that it's Intel-specific (Sapphire Rapids+, no AMD equivalent yet) and the producer has to know it's about to publish. For streaming inter-thread workloads, it's a major lever; for general code, it doesn't help.

## Why thread-per-core works

By pinning one thread per physical core with its own allocator, network queue, and storage partition, [[thread-per-core]] eliminates all inter-core cache coherency traffic. [[seastar|ScyllaDB's Seastar]] proved this enables near-linear scaling. The Rust [[thread-per-core]] runtimes — [[monoio]], [[compio]], [[glommio]] — achieve 2–3× [[tokio]] throughput at 16+ cores primarily by avoiding these costs.

## Cache-friendly data structure design

Cache coherency costs are one dimension; cache *utilization* is the other. The [[soa-vs-aos]] pattern (1.4–12.7× speedup), [[b-tree]] over red-black trees (3–18×), and [[csr-graph|CSR]] over adjacency lists (40–250×) all demonstrate that contiguous memory layouts exploiting cache lines dominate pointer-chasing alternatives. The [[lmax-disruptor]] explicitly pads sequence counters to cache-line boundaries to prevent false sharing — and the modern correction to that practice is [[false-sharing|128-byte padding rather than 64-byte]], because x86's adjacent cache-line prefetcher pulls in the neighbor on every fetch. The 64→128 fix alone is worth ~1.7× on contended ring buffers. See [[fastest-data-structures]] for the full benchmarked landscape.

## See also

- [[memory-ordering]] — TSO vs relaxed models layered on top of coherence
- [[false-sharing]] — the most common coherence-cost violation
- [[inter-thread-communication]] — the broader latency ladder
- [[core-pinning]] — exploiting topology
- [[mechanical-sympathy]] — designing with coherence in mind
