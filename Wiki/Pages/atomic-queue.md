---
tags:
  - cpp
  - concurrency
  - performance
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# atomic_queue

`atomic_queue` is a header-only C++ library by Maxim Egorshin providing several lock-free MPMC queue implementations. Its **`OptimistAtomicQueue`** consistently wins C++ benchmarks for **highest throughput and lowest latency**: sub-100 ns round-trip push-to-pop in MPMC mode, SPSC throughput exceeding 200M ops/s, and 200–500M+ ops/s in 1P/1C configuration.

## What's inside

The library provides a family of fixed-capacity, [[ring-buffer|ring-buffer]]-backed queues:

- `AtomicQueue` — the conservative MPMC implementation (CAS-retry on a per-slot state)
- `OptimistAtomicQueue` — the headline, optimized hot path; busy-waits on slot state
- `AtomicQueueB` / `OptimistAtomicQueueB` — bounded variants using element-as-sentinel
- SPSC specializations that drop the atomic RMW entirely

All use power-of-two capacity for cheap modular arithmetic and apply [[false-sharing|cache-line padding]] aggressively.

## Why it wins on latency

`OptimistAtomicQueue` is engineered for the *uncontended* case: when the next slot is in the expected state, the operation completes in two atomic loads plus a release store, no RMW. Under contention it falls back to a brief busy-wait. This makes it ideal for low-latency systems where the queue is rarely actually contended (HFT, real-time DSP, telemetry pipelines) but absolutely needs sub-100 ns latency on the common path.

## When to use it vs alternatives

- **`atomic_queue`** — lowest latency, strict FIFO, x86 and ARM. The default for new C++ code that needs raw speed.
- **[[moodycamel-concurrent-queue]]** — higher absolute throughput in many-producer scenarios, but no strict FIFO across producers.
- **[[lcrq|LCRQ]] C++ port** (Pedro Ramalhete) — higher MPMC throughput at very high thread counts on x86, but more complex and CAS2-dependent.

## See also

- [[mpmc-queue]] — broader landscape
- [[spsc-queue]] — SPSC tier
- [[the-fastest-queue]] — full hierarchy
