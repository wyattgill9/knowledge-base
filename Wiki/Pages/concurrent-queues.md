---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Concurrent Queues

High-performance lock-free and low-lock queue implementations for inter-thread communication. The [[lmax-disruptor]] (52 ns/hop) remains the latency king for SPMC pipelines, but several MPMC alternatives serve different trade-off points.

## MPMC queue landscape

**LCRQ (Linked Concurrent Ring Queue)** (Morrison & Afek) — fastest MPMC on x86, using fetch-and-add for slot assignment with SIMD-friendly ring buffer arrays. Drawbacks: requires double-word CAS (limiting portability) and can consume ~400 MB under load.

**FAAArrayQueue** (Pedro Ramalhete) — nearly matches LCRQ throughput with only 8 bytes per entry and standard CAS. The portable alternative when LCRQ's double-word CAS is unavailable.

**moodycamel::ConcurrentQueue** (C++, 18k+ GitHub stars) — 1.7x faster than Boost.Lockfree through per-producer implicit sub-queues that eliminate inter-producer contention. A practical, well-tested choice for C++ applications.

**ParlayHash** (CMU) — while technically a hash map, its 1,130 million ops/sec at 128 threads (vs 29 Mops/s for libcuckoo) demonstrates the state of the art in concurrent data structure throughput.

## In Rust

[[kanal]] leads benchmarks at 8–16M msg/sec for message passing. [[crossbeam-channel]] provides proven synchronous MPMC channels. For concurrent maps rather than queues, [[papaya]] (lock-free, read-optimized), [[dashmap]] (sharded RwLock, best write throughput), and [[scc]] (extreme write contention) cover different profiles.

## Lock-free stacks

The **Elimination-Backoff Stack** (Hendler, Shavit, Yerushalmi) dramatically outscales the classic Treiber stack at high thread counts by matching complementary push/pop operations in a randomized elimination array, bypassing the central head-pointer bottleneck. At 64 threads with 50/50 push/pop: 100K+ ops/msec.
