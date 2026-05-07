---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# LMAX Disruptor

The Disruptor is the gold standard for **bounded pipeline latency**, created by Martin Thompson et al. at LMAX Exchange. Its pre-allocated [[ring-buffer]] with sequence-number-based coordination achieves **52 ns mean latency per hop** and **128 ns at the 99th percentile** — compared to 32,757 ns mean and 2,097,152 ns p99 for Java's `ArrayBlockingQueue`. That is **630× lower mean latency** and **8× higher throughput** (26M vs 5.3M ops/s in 1P/1C). LMAX itself processes 6 million orders per second on a single JVM thread.

## Where the Disruptor sits in 2025

The Disruptor is **not** the absolute throughput champion — for raw MPMC throughput, [[lcrq|LCRQ]] + [[aggregating-funnels]] (PPoPP 2025) wins by a wide margin, and a fully-optimized C++ [[spsc-queue|SPSC ring buffer]] hits 200–500M ops/s versus the Disruptor's 26M. What the Disruptor wins is **latency tail** in bounded pipeline topologies: pre-allocated objects, single-writer principle, mechanical-sympathy padding, and the ability for multiple consumers to read the same event without locking.

| Use case | Best choice |
|----------|-------------|
| Lowest tail latency in a bounded pipeline | **LMAX Disruptor** |
| Max strict-FIFO MPMC throughput on x86 | [[lcrq]] + [[aggregating-funnels]] |
| Max strict-FIFO MPMC, portable | [[scq]] / [[lprq]] |
| Wait-free MPMC | [[wcq]] |
| One-producer, one-consumer raw throughput | [[spsc-queue]] (rtrb, atomic_queue) |
| C++ MPMC, relaxed FIFO | [[moodycamel-concurrent-queue]] |

## Design: mechanical sympathy

The Disruptor embodies "mechanical sympathy" — designing software that works with the hardware rather than against it:

- **Pre-allocated ring buffer** — no allocation on the hot path, no GC pressure. Sized to a power of 2 so index computation is bitwise AND.
- **Single-writer principle** — each sequence number is written by exactly one thread, eliminating write contention. Multiple consumers read the same buffer; they coordinate via published sequence numbers, not by claiming slots competitively. This is exactly the [[spsc-queue|SPSC]] privilege extended to a one-producer-many-consumer pipeline.
- **[[false-sharing|Cache-line padding]]** — each sequence counter is padded to fill a cache line, preventing false sharing. The original Disruptor used 64-byte padding; modern Java reimplementations use 128 bytes to defeat the adjacent cache-line prefetcher, capturing the additional ~1.7×.
- **Sequence-number coordination** — no locks, no CAS on the critical path. Producers and consumers coordinate by reading each other's monotonically-increasing sequence numbers.

## Latency comparison

| Metric | ArrayBlockingQueue | LMAX Disruptor |
|--------|-------------------|----------------|
| Throughput (1P/1C) | 5.3M ops/s | **26.0M ops/s** |
| Mean latency | 32,757 ns | **52 ns** |
| 99th percentile latency | 2,097,152 ns | **128 ns** |

The 16,000× p99 advantage is the headline. ArrayBlockingQueue's tail is dominated by lock contention and GC pauses; the Disruptor has neither.

## When to use it

The Disruptor pattern is ideal when:

- The pipeline topology is fixed (known producers, known consumers, known stages)
- Tail latency matters more than peak throughput
- The buffer can be pre-allocated (bounded queue, no dynamic resizing)
- Multiple consumers need to read the same events (broadcast pipelines)

It is *not* the best choice when:

- The topology is dynamic (unknown number of producers/consumers)
- Memory is tight (pre-allocation forces fixed footprint)
- Strict cross-producer FIFO across many producers is needed at maximum throughput — for that, [[lcrq|LCRQ]] / [[scq|SCQ]] win

For read-dominated workloads, [[rcu|RCU]] achieves even lower overhead — effectively zero on the read side.

## Implementations

- **Java** — the original LMAX Disruptor library; production-deployed across financial-services systems
- **C++** — multiple ports, reaching 50–95 million ops/sec
- **Rust** — `disrustor` and similar; idiomatic Rust treatment of sequence-based coordination
- **The pattern itself** — the single-writer + sequence-coordination idea has been re-implemented in nearly every language; what's distinctive is the discipline, not the code

## See also

- [[ring-buffer]] — the underlying structure
- [[spsc-queue]] — same fundamental privilege, two-thread variant
- [[false-sharing]] — the 128-byte rule the Disruptor's padding is a poster child for
- [[the-fastest-queue]] — broader hierarchy
- [[concurrent-queues]] — broader landscape
