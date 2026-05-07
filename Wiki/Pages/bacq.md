---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# BACQ (Boundary-Aware Concurrent Queue)

BACQ (2024) is the current state-of-the-art GPU queue, achieving **2–9× improvement over previous GPU queue designs** including warp-centric LCRQ ports. Its contribution is treating the boundary between full and empty queue states as a first-class concern rather than a special case.

## The boundary insight

Most GPU queue performance papers focus on the steady-state throughput when the queue is comfortably non-empty. In reality, GPU workloads frequently push the queue against full or empty boundaries — when many threads simultaneously enqueue (filling the buffer) or dequeue (draining it). At those moments, contention on the size counter and the wrap-around logic dominates.

BACQ explicitly models the queue's distance from boundary states and uses different code paths near boundaries:

- **Steady state** — warp-aggregated FAA on the index, fast path.
- **Near boundary** — explicit synchronization to handle the transition, batch overflow/underflow detection across the warp.

The 2–9× improvement comes mostly from the latter — the previous-best designs handled boundary transitions one thread at a time, serializing whole warps.

## Position

For [[gpu-queues|GPU queues]], BACQ is the current ceiling. For CPU queues, the equivalent role is filled by [[lcrq|LCRQ]] + [[aggregating-funnels]]. The two architectures genuinely require different algorithms — a hardware divergence that has only widened as GPU thread counts have grown into the hundreds of thousands.

## See also

- [[gpu-queues]] — broader GPU queue context
- [[lcrq]] — CPU-side counterpart
- [[the-fastest-queue]] — full hierarchy
