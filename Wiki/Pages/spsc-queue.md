---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# SPSC Queue

A single-producer single-consumer queue is the fastest concurrent queue achievable, because the topology eliminates the need for any atomic read-modify-write instruction. Acquire loads and release stores — the cheapest atomics on every modern architecture — are sufficient to coordinate one producer and one consumer over a [[ring-buffer|ring buffer]]. Optimized SPSC queues hit **300–530 million operations per second**: a Rust `ringbuffer-spsc` benchmark on Apple M4 reaches **520–530M ops/s**. The design lineage runs from **Lamport's 1977 correctness proof** through Erik Rigtorp's modern cached-indices implementation, with every fast queue in the last decade descending from the same insight.

## The privileged tier

| Implementation | Throughput | Hardware |
|----------------|------------|----------|
| Naïve SPSC ring buffer (seq_cst) | 5.5M ops/s | AMD Ryzen 9 3900X |
| Optimized SPSC (acquire/release, cached indices) | 112M ops/s | AMD Ryzen 9 3900X |
| Fully optimized C++ SPSC | **305M ops/s** | Intel Core Ultra 5 135U |
| Rust [[rtrb|ringbuffer-spsc]] | **520–530M ops/s** | Apple M4 |

The 20× gap between naïve and optimized on identical hardware is almost entirely cache coherence reduction, not algorithmic differences.

## Three optimizations that compound

**1. Acquire/release instead of seq_cst.** Sequential consistency forces a full memory fence on every atomic store, costing 30–60 ns of pipeline stall on x86 and *significantly* more on ARM. Release stores and acquire loads are the minimum sufficient ordering for SPSC: the producer's release pairs with the consumer's acquire so the consumer sees a complete write, but the rest of the world's view is unconstrained. See [[memory-ordering]] for why this is free on x86 and not free on ARM.

**2. [[false-sharing|128-byte padding between head and tail]].** Without padding, every producer write to `tail` invalidates the consumer's `head` cache line, and vice versa. The 64-byte rule is not enough — the adjacent cache-line prefetcher pulls in the neighbor on every fetch. Padding to 128 bytes delivers roughly 3.1× over unpadded.

**3. Shadow variables (cached remote indices).** The killer optimization, popularized by **Erik Rigtorp's SPSCQueue**. The producer keeps a *local cached copy* of the consumer's `head` index and only re-reads it when its cached value would indicate the queue is full. Symmetrically for the consumer's view of `tail`. This converts per-operation cache-line transfers (~3 per op) into per-batch transfers (~1 per N ops), and is the dominant source of the 20× speedup. Rigtorp's measurement on AMD Ryzen 9 3900X: 5.5M ops/s naive → 112M ops/s with cached indices, a single-optimization gain larger than any other queue technique.

```rust
// Producer side, simplified
fn push(&mut self, val: T) -> Result<(), T> {
    let tail = self.tail.load(Relaxed);
    let next = (tail + 1) & self.mask;
    if next == self.cached_head {           // fast path: use stale cached head
        self.cached_head = self.head.load(Acquire);  // refresh cache
        if next == self.cached_head {
            return Err(val);                 // really full
        }
    }
    unsafe { self.buffer[tail].write(val); }
    self.tail.store(next, Release);
    Ok(())
}
```

The `cached_head` lives in the producer's L1 and updates only on the slow path. In steady state, almost every push is a single load + store + plain memory write.

## Why no read-modify-write is needed

A `LOCK XADD` (FAA) or `LOCK CMPXCHG` (CAS) costs ~10 ns *uncontended* and 40–75 ns under heavy contention because of the cache coherence protocol. SPSC queues sidestep these entirely: the producer is the only writer of `tail`, the consumer is the only writer of `head`. There is exactly one writer per atomic, so a plain release store works — no atomic RMW, no LOCK prefix.

This is why SPSC throughput is in a different league from MPMC. Going from one producer to two producers requires FAA or CAS on `tail`, which is the dominant cost. SPSC's privileged status is a topology fact, not a clever algorithm. Lamport proved the correctness of this in 1977; everything since has been engineering against the cache-coherence floor.

## Backing the buffer with huge pages

For ring buffers larger than ~256 KB, [[huge-pages|huge-page backing]] eliminates TLB misses across the buffer. A 4 MB SPSC ring backed by 4 KB pages needs ~1024 TLB entries to cover; the same buffer on a 2 MB huge page needs 2. Combined with [[core-pinning|`taskset` pinning]] to cores sharing an L3 and [[busy-spin|busy-spin consumers]], this is the [[inter-thread-communication|HFT recipe]] that approaches the bare cache-coherence floor.

## Per-language fastest

- **C++** — `atomic_queue` (`OptimistAtomicQueue`) and `folly::ProducerConsumerQueue`; the latter at ~67M ops/s, the former 200M+.
- **Rust** — [[rtrb|`rtrb`]], `bounded-spsc-queue`, `ringbuffer-spsc`; ~7 ns per op, 520M+ on Apple M4.
- **Java** — `JCTools` `SpscArrayQueue` and the [[lmax-disruptor]] in 1P/1C mode (26M ops/s, 52 ns mean latency).
- **C** — Concurrency Kit's `ck_ring`.

## When to use SPSC

When you can guarantee exactly one producer and one consumer, always. The throughput gap to MPMC is 5–50×, and the topology constraint is usually easy to engineer (per-pipeline-stage queues, per-worker mailboxes, per-CPU log buffers in [[thread-per-core|thread-per-core]] runtimes). When you cannot, the next best option is the [[lmax-disruptor]] (which extends SPSC's logic to multi-consumer pipelines via the single-writer principle) or [[lcrq|LCRQ]] for true MPMC.

## See also

- [[ring-buffer]] — the underlying structure
- [[mpmc-queue]] — what you give up when you need multi-producer
- [[lmax-disruptor]] — single-writer extension to multi-consumer pipelines
- [[false-sharing]] — the 128-byte rule
- [[memory-ordering]] — why acquire/release is free on x86
- [[busy-spin]] — the wait strategy that pairs with SPSC
- [[inter-thread-communication]] — broader latency hierarchy
- [[the-fastest-queue]] — full hierarchy
