---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# LMAX Disruptor

The gold standard for inter-thread communication latency, created by Martin Thompson et al. at LMAX Exchange. Its pre-allocated ring buffer with sequence-number-based coordination achieves **52 ns per hop** — compared to 32,757 ns for Java's `ArrayBlockingQueue`, a **630x improvement**. The C++ port reaches 50–95 million ops/sec.

## Design: mechanical sympathy

The Disruptor embodies "mechanical sympathy" — designing software that works with the hardware rather than against it:

- **Pre-allocated ring buffer** — no allocation on the hot path, no GC pressure. The buffer is sized to a power of 2, making index computation a bitwise AND.
- **Cache-line padding** — each sequence counter is padded to fill an entire cache line, preventing false sharing between producers and consumers.
- **Single-writer principle** — each sequence number is written by exactly one thread, eliminating write contention and the [[cache-coherency|MESI protocol]] costs of shared cache lines.
- **Sequence-number coordination** — no locks, no CAS on the critical path. Producers and consumers coordinate by reading each other's sequence numbers, which are monotonically increasing.

## Comparison with alternatives

For MPMC (multi-producer, multi-consumer) queues, several alternatives exist:

**LCRQ** (Morrison & Afek) is the fastest MPMC queue on x86, using fetch-and-add for slot assignment. However, it requires double-word CAS (limiting portability) and can consume ~400MB under load.

**FAAArrayQueue** (Pedro Ramalhete) nearly matches LCRQ throughput with only 8 bytes per entry and standard CAS — the portable alternative.

**moodycamel::ConcurrentQueue** (18k+ GitHub stars) provides 1.7x faster throughput than Boost.Lockfree through per-producer implicit sub-queues that eliminate inter-producer contention. In Rust, [[kanal]] leads benchmarks at 8–16M msg/sec for message passing.

## Lock-free stacks

The **Elimination-Backoff Stack** (Hendler, Shavit, Yerushalmi) dramatically outscales the classic Treiber stack: at 64 threads with 50/50 push/pop, it achieves 100K+ ops/msec by matching complementary operations in a randomized elimination array, bypassing the central head-pointer bottleneck.

## When to use the Disruptor pattern

The Disruptor pattern is ideal when you have a fixed pipeline topology (known producers and consumers), need minimal latency between stages, and can pre-allocate buffer space. It is less suitable for dynamic topologies or when memory is constrained. For read-dominated workloads, [[rcu]] achieves even lower overhead — effectively zero on the read side.
