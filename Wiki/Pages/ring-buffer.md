---
tags:
  - data-structures
  - performance
  - concurrency
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Ring Buffer

A ring buffer (circular buffer) is a fixed-size array with head and tail indices that wrap modulo capacity. For single-threaded FIFO queuing, it achieves the theoretical lower bound of Ω(1) per operation with the lowest constant factors available, **roughly 1.5× faster** than C++ `std::queue` and **2–10× faster** than linked-list queues once the working set exceeds L1. Every fast queue in modern systems — [[spsc-queue|SPSC]], [[lmax-disruptor|LMAX Disruptor]], [[lcrq|LCRQ]] — is a ring buffer at its core.

## Why it dominates

- **Contiguous memory** — the array fits in cache lines, the prefetcher does the right thing for free, and there is no pointer chasing.
- **Bitwise wrap** — when capacity is a power of two, `index & (capacity - 1)` replaces an expensive modulo with a single AND instruction.
- **Zero allocation** — the buffer is allocated once at construction; enqueue and dequeue are pure index arithmetic plus a load/store.
- **Bounded** — by design. Backpressure is automatic; no GC, no fragmentation, no OOM.

The compiled hot path is a handful of instructions on every modern architecture.

## Linked-list queues lose, period

A linked-list queue allocates a node per enqueue. This kills performance on three axes:

1. **Cache locality** — nodes are scattered across the heap; the prefetcher cannot follow pointer chains, so every dequeue stalls on a cache miss for queues larger than L1.
2. **Allocation cost** — 50–100+ ns per node even with a fast allocator like [[jemalloc]] or [[mimalloc]]. For a 10 ns memory operation, that is overhead measured in *thousands* of percent.
3. **Memory overhead** — each node carries a `next` pointer (8 bytes) and allocator metadata (8–16 bytes more) per element.

For queues exceeding L1 cache size, linked-list traversal runs **2–10× slower** than contiguous structures. The only things linked-list queues are good for are concurrent designs where the link itself enables fine-grained synchronization (the [[michael-scott-queue]] uses links to swing tail pointers atomically) — and even those have been displaced by FAA-on-a-ring designs.

## The unbounded case: chunked / unrolled lists

When unboundedness is required, the right structure is a **chunked linked list**: one node holds many elements (typically a cache line's worth, 8 pointers or 16 ints). This recovers most of the ring buffer's cache performance — up to **60% fewer cache misses** than per-element nodes — while supporting unbounded growth.

C++'s `std::deque` is essentially a less-optimized chunked linked list. A growable ring buffer (doubling on overflow) provides amortized O(1) with even better cache behavior, at the cost of O(n) on resize.

## The virtual-memory trick

A ring buffer's only branch in the hot path is the wrap check. You can eliminate it by mapping the same physical buffer pages **twice contiguously** in virtual address space: the kernel sees one buffer; the program sees two copies back-to-back. Reads and writes near the boundary span the seam without a branch because the second copy is identical to the first.

This trick was popularized in audio DSP and pcap ring buffers; it is essentially free on Linux via `mmap(MAP_ANONYMOUS)` plus `mremap`, and on Windows via `MapViewOfFileEx`. The savings are small (one branch eliminated) but in tight loops processing samples or packets the branch is on the hot path.

## Concurrent ring buffers

The single-threaded ring buffer's properties survive into the concurrent setting if you can keep the head and tail in separate cache lines:

- **[[spsc-queue|SPSC ring buffer]]** — only two threads touch the structure; acquire/release atomics suffice. 300–530M ops/s.
- **[[lmax-disruptor]]** — single-writer principle; sequence numbers coordinate without CAS on the hot path. 26M ops/s, 52 ns mean latency.
- **[[lcrq|LCRQ]]** — multiple producers/consumers; FAA on shared indices, slot-level CAS. The fastest MPMC.

The crucial discipline in all three is **[[false-sharing|128-byte cache-line isolation]]** between head and tail, which delivers roughly 3× over unpadded queues by defeating the adjacent cache-line prefetcher.

## See also

- [[spsc-queue]] — single-producer single-consumer optimizations
- [[mpmc-queue]] — multi-producer multi-consumer landscape
- [[lcrq]], [[lmax-disruptor]] — concurrent ring buffer designs
- [[the-fastest-queue]] — full hierarchy
- [[false-sharing]] — why the 128-byte rule matters
