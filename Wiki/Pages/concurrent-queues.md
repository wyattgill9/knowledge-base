---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Concurrent Queues

The landscape of high-performance concurrent FIFO queues. Three primitive choices dominate every design: **[[faa-vs-cas|FAA over CAS]]** at contention hotspots, **[[ring-buffer|ring buffers]] over linked lists** for cache locality, and **[[false-sharing|128-byte cache-line isolation]]** for head/tail indices. Combined, these account for the entire 5–10× gap between modern lock-free queues and the [[michael-scott-queue|1996 Michael-Scott baseline]] still in many standard libraries.

For full architectural context, see [[the-fastest-queue]]. For the Disruptor specifically, see [[lmax-disruptor]].

## The MPMC hierarchy (2025)

| Algorithm | Year | Property | Position |
|-----------|------|----------|----------|
| [[lcrq]] + [[aggregating-funnels]] | 2025 | x86, lock-free | **Throughput ceiling** |
| [[lcrq]] | 2013 | x86, lock-free | Original FAA-on-a-ring |
| [[lprq]] | 2023 | Portable, lock-free | Direct LCRQ port without CAS2 |
| [[scq]] | 2019 | Portable, lock-free | Half the memory of LCRQ; threshold mechanism |
| [[wcq]] | 2022 | Portable, **wait-free** | Fast-path-slow-path; matches SCQ throughput |
| [[moodycamel-concurrent-queue]] | C++ | Relaxed FIFO | Per-producer sub-queues; ~1.7× Boost.Lockfree |
| [[atomic-queue]] | C++ | Strict FIFO | Sub-100 ns latency, 200–500M+ ops/s 1P/1C |
| [[michael-scott-queue]] | 1996 | Lock-free, linked | Historical baseline, **3× slower than LCRQ** |

## SPSC and the topology privilege

[[spsc-queue|Single-producer single-consumer]] queues occupy a separate tier — **300–530M ops/s** — because they need only acquire/release atomics, no atomic RMW. The optimization stack: 128-byte padding, shadow variables (cached remote indices), acquire/release rather than seq_cst, power-of-two ring with bitwise wrap. A naive SPSC at 5.5M ops/s and an optimized SPSC at 530M ops/s use the *same algorithm* on the *same hardware* — the 100× gap is entirely cache coherence engineering.

## The Rust ecosystem

| Need | Crate | Notes |
|------|-------|-------|
| Bounded MPMC queue | [[crossbeam-array-queue\|crossbeam ArrayQueue]] | Vyukov-style ring; no native LCRQ port exists |
| Unbounded MPMC queue | `crossbeam::queue::SegQueue` | Chunked linked list with [[crossbeam-epoch]] |
| Sync MPMC channel | [[crossbeam-channel]] | Standard for sync message-passing |
| Async / max throughput | [[kanal]] | 8–16M msg/sec, unified sync/async |
| Within Tokio | `tokio::sync::mpsc` | Intra-thread coroutine switching |
| Wait-free SPSC | [[rtrb]] | ~7 ns/op, 520M+ on Apple M4 |

## The C++ ecosystem

| Need | Library | Notes |
|------|---------|-------|
| Lowest latency MPMC | [[atomic-queue]] | OptimistAtomicQueue; sub-100 ns round-trip |
| General MPMC | [[moodycamel-concurrent-queue]] | ~114M ops/s; relaxed cross-producer FIFO |
| Strict-FIFO max throughput | [[lcrq]] (Pedro Ramalhete port) | x86 only |
| Bounded pipelines | [[lmax-disruptor]] | 26M ops/s, 52 ns mean latency |

## Specialized contexts

- **[[numa-aware-queues|NUMA]]** — Nuddle delegation, SmartPQ; 1.87× over NUMA-oblivious on 2+ socket systems.
- **[[gpu-queues|GPU]]** — warp-centric design mandatory; CAS-based queues 1,112× slower than blocking array queues at high thread counts; [[bacq|BACQ]] is current state of the art.
- **[[persistent-queues|Persistent (functional)]]** — Hood-Melville (1981), Okasaki (1995), Kaplan-Tarjan (1999) for worst-case O(1); the Kaplan-Tarjan deque was finally verified in OCaml in 2025.

## Lock-free stacks

The [[elimination-backoff-stack|Elimination-Backoff Stack]] (Hendler, Shavit, Yerushalmi 2004) dramatically outscales the classic Treiber stack at high thread counts by matching complementary push/pop operations in a randomized elimination array, bypassing the central head-pointer bottleneck. At 64 threads with 50/50 push/pop: 100K+ ops/msec.

## Concurrent maps (adjacent landscape)

For concurrent *maps* rather than queues: [[papaya]] (lock-free, read-optimized), [[dashmap]] (sharded RwLock, best write throughput), [[scc]] (extreme write contention) cover Rust profiles. CMU's [[parlayhash]] hits 1,130 Mops at 128 threads — 39× libcuckoo — and represents the C++ state of the art. See [[fastest-hash-map-2025]] and [[rust-concurrent-data-structures]].
