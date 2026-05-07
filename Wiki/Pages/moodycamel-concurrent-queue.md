---
tags:
  - cpp
  - concurrency
  - performance
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-05-06
---

# moodycamel::ConcurrentQueue

A widely-deployed C++ MPMC queue (18k+ GitHub stars) by Cameron Desrochers. Its performance comes from **per-producer implicit sub-queues** that eliminate inter-producer contention: each producer enqueues to its own sub-queue, and consumers round-robin across sub-queues. This trades **strict cross-producer FIFO ordering** — which is rarely actually needed — for **~1.7× throughput over Boost.Lockfree** and competitive performance across the C++ MPMC landscape.

## Design

When a thread first enqueues, it gets allocated an implicit producer slot. That producer's sub-queue is essentially [[spsc-queue|SPSC]] — only that thread writes to it — so per-producer push is fast and uncontended. Consumers iterate sub-queues looking for a non-empty one and pop. The result:

- ~114M ops/s single-threaded; scales well with producers
- Sub-100 ns push and pop in the common case
- No strict FIFO across producers (a later push by producer A may be consumed before an earlier push by producer B)

For most workloads — task queues, work distribution, logging — relaxed FIFO is fine. For workloads where strict ordering across producers is required (e.g., serializing requests for a state machine), prefer [[lcrq|LCRQ]], [[scq|SCQ]], or [[atomic-queue|atomic_queue]].

## Comparison

| Queue | Throughput | Strict FIFO | Notes |
|-------|-----------|-------------|-------|
| [[atomic-queue\|atomic_queue]] | 200–500M+ ops/s 1P/1C | Yes | Lowest latency, sub-100 ns round-trip |
| [[lcrq]] (C++ port) | Higher MPMC, x86 only | Yes | CAS2-dependent |
| moodycamel::ConcurrentQueue | ~114M ops/s 1P/1C | **No** | Best general-purpose C++ MPMC |
| `boost::lockfree::queue` | ~1.7× slower | Yes | Linked-list, Michael-Scott style |

## When to choose it

- General-purpose C++ MPMC where strict cross-producer FIFO is not required
- Many producers known to outnumber consumers (the implicit-producer model amortizes well)
- Pre-existing C++ codebase looking for a drop-in replacement for `std::queue` + mutex

For lower latency or strict FIFO, [[atomic-queue]]. For x86-only max throughput, the C++ port of [[lcrq]].

## See also

- [[mpmc-queue]] — broader landscape
- [[atomic-queue]] — strict-FIFO C++ alternative
- [[lcrq]] — the throughput ceiling
- [[concurrent-queues]]
