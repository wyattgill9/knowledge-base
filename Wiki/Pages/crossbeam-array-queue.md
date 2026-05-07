---
tags:
  - rust
  - concurrency
  - data-structures
  - crate
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# crossbeam ArrayQueue

`crossbeam::queue::ArrayQueue` is the standard bounded lock-free [[mpmc-queue|MPMC]] queue in Rust. It is a Vyukov-style ring buffer (Dmitry Vyukov, 2010) — a fixed-capacity [[ring-buffer]] with per-slot sequence numbers — predating the [[lcrq|LCRQ]] family but holding up well in practice.

## How it works

Each slot stores a `(sequence, value)` pair. Producers spin-wait for the slot's sequence to match the producer index before claiming it via CAS; consumers spin-wait for the slot's sequence to match the consumer index plus one. This avoids ABA without epoch reclamation and gives strict FIFO across all producers and consumers.

The price is CAS retries on slot acquisition under heavy contention — not as bad as Michael-Scott (which retries on a single contended pointer) but worse than [[lcrq|LCRQ]]'s FAA-on-an-index. For Rust's typical use cases (small numbers of producers/consumers, dozens not hundreds), the difference is rarely material.

## Position in the Rust ecosystem

| Need | Use |
|------|-----|
| Bounded MPMC queue | `crossbeam::queue::ArrayQueue` |
| Unbounded MPMC queue | `crossbeam::queue::SegQueue` (chunked linked list) |
| Synchronous MPMC channel | [[crossbeam-channel]] |
| Async / max-throughput channel | [[kanal]] |
| Wait-free SPSC | [[rtrb]] |

No native [[lcrq|LCRQ]], [[scq|SCQ]], or [[lprq|LPRQ]] port exists in Rust as of 2025. `ArrayQueue` is the closest production-ready equivalent for bounded MPMC, and it's good enough that the absence has not been pressing.

## See also

- [[mpmc-queue]] — broader landscape
- [[crossbeam-epoch]] — the reclamation primitive used by `SegQueue`
- [[ring-buffer]] — underlying structure
