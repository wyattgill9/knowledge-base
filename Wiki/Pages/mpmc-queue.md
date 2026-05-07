---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# MPMC Queue

A multi-producer multi-consumer queue is what most concurrency papers mean when they say "concurrent queue." Unlike [[spsc-queue|SPSC]], which gets to skip atomic RMW entirely, MPMC queues need at least one atomic operation per push and pop on a contended index — and that single instruction choice is the dominant performance lever. The 2025 hierarchy: [[lcrq|LCRQ]] + [[aggregating-funnels]] is the throughput ceiling, [[scq|SCQ]] and [[lprq|LPRQ]] are the portable equivalents, [[wcq|wCQ]] adds wait-freedom essentially for free, and [[michael-scott-queue]] is the historical baseline that everything else beats.

## The throughput tiers

| Algorithm | Year | Property | Relative throughput |
|-----------|------|----------|---------------------|
| [[lcrq]] + [[aggregating-funnels]] | 2025 | x86, lock-free | **1.0× (baseline / ceiling)** |
| [[lcrq]] | 2013 | x86, lock-free | ~0.4× |
| [[lprq]] | 2023 | Portable, lock-free | ~0.4× (matches LCRQ) |
| [[scq]] | 2019 | Portable, lock-free | ~0.4×, half the memory |
| [[wcq]] | 2022 | Portable, **wait-free** | ~0.4× (matches SCQ) |
| Flat-combining queue | 2010 | Lock-free | ~0.16× |
| [[michael-scott-queue]] | 1996 | Lock-free, linked | ~0.13× |

The factor of ~7× between Michael-Scott and LCRQ+Funnels is essentially the cumulative effect of: FAA over CAS (1.8×), FAA-then-slot-CAS pattern over CAS-retry (~2×), and software-combined FAA over plain FAA (up to 2.5×).

## What separates a fast MPMC from a slow one

**[[faa-vs-cas|FAA on the contended index]].** This is the most important decision. Michael-Scott uses CAS-retry on head/tail pointers and melts down past 32 threads. LCRQ uses a single FAA on an integer slot index that always succeeds — throughput plateaus instead of collapsing.

**Slot-level synchronization separately from index assignment.** After FAA gives a thread its unique slot, the actual data exchange happens locally on that slot — uncontended. LCRQ uses 128-bit CAS for this; SCQ and LPRQ use single-width CAS plus a threshold or auxiliary state.

**[[false-sharing|128-byte cache-line isolation]] of head and tail.** A 1.7× free win on x86 from defeating the adjacent-cache-line prefetcher.

**Memory reclamation.** Lock-free queues that allocate (Michael-Scott, FAAArrayQueue) need epoch-based reclamation ([[crossbeam-epoch]]) or hazard pointers to avoid use-after-free. The Cyclic Memory Protection (CMP) preprint from November 2025 claims coordination-free reclamation at 6.49M items/s but is not yet peer-reviewed.

## The trade-off knobs

- **x86 vs portable.** LCRQ uses `CMPXCHG16B`. SCQ and LPRQ run everywhere. If you only target x86, plain LCRQ is a fine choice; otherwise SCQ or LPRQ.
- **Bounded vs unbounded.** All the FAA-ring queues are linked lists of fixed-size rings, so they can grow. Bounded variants (SCQ on a single ring, atomic_queue) achieve O(1) worst-case with no allocation.
- **Strict FIFO vs relaxed FIFO.** [[moodycamel-concurrent-queue]] sacrifices strict cross-producer ordering for higher throughput via per-producer sub-queues. If you don't need strict FIFO across producers, this is competitive.
- **Lock-free vs wait-free.** wCQ shows wait-freedom is no longer expensive. For hard real-time systems, prefer wCQ; otherwise lock-free SCQ/LCRQ is fine.

## In practice (per language)

- **C++** — [[atomic-queue]] (`OptimistAtomicQueue`) for max throughput, [[moodycamel-concurrent-queue]] for general use, `boost::lockfree::queue` for simple cases.
- **Rust** — [[crossbeam-array-queue|crossbeam `ArrayQueue`]] for bounded MPMC, [[crossbeam-channel]] for synchronous message-passing, [[kanal]] for max raw throughput. No native LCRQ port exists.
- **Java** — JCTools `MpmcArrayQueue` (~65M ops/s), the [[lmax-disruptor]] for bounded pipelines.
- **Go** — Lock-free ring buffers narrow the gap with C++ to 3–5×; channels are 10–50× slower than optimized C++.

## See also

- [[lcrq]], [[scq]], [[lprq]], [[wcq]] — the 2013–2025 algorithm progression
- [[aggregating-funnels]] — the 2025 throughput multiplier
- [[michael-scott-queue]] — the historical baseline
- [[the-fastest-queue]] — full hierarchy
- [[spsc-queue]] — what you get when topology lets you skip MPMC
