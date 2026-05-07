---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# GPU Queues

GPUs run **160,000+ concurrently active threads**, have no coherent L1 caches, and execute in SIMT (Single Instruction Multiple Threads) lockstep across 32-thread warps. Every assumption built into a CPU lock-free queue breaks. CAS-based queues melt down catastrophically — one published benchmark shows a CAS-based queue running **1,112× slower** than even a blocking array queue at high thread counts. The right answer is **warp-centric** queue design.

## Why CPU queues don't translate

- **No coherent L1.** Threads in different warps may not see each other's writes without explicit fences. CAS retry loops degrade because the contended cache line has to ping back to L2 between every retry.
- **SIMT execution.** All 32 threads in a warp execute the same instruction. A CAS retry loop where some threads succeed and others fail divergently serializes the warp — the failing threads stall the whole warp until they catch up.
- **Massive thread count.** A queue contended by 100,000 threads has nothing in common with a CPU queue contended by 100. Hardware FAA hotspots saturate immediately.

## Warp-centric design

The dominant pattern aggregates operations within a warp before touching shared state:

1. All 32 threads in a warp ballot whether they want to enqueue/dequeue.
2. The warp leader performs **one** atomic claim for all participating threads' slots.
3. Threads write/read their respective slots in parallel.

Up to **40× speedup** over thread-centric designs in published benchmarks. The throughput regime is genuinely different: an [[lcrq|LCRQ]]-family queue ported to NVIDIA K20c reached **201.8 million ops/s at 623 threads**, an order of magnitude below CPU peak but on a vastly different parallelism model.

## BACQ (2024)

[[bacq|Boundary-Aware Concurrent Queue]] (BACQ, 2024) is the current state of the art for GPU queues, achieving **2–9× improvement over existing GPU queue designs**. Its key insight is that GPU queue performance is dominated by the *boundary* between full and empty states — most contention happens when the queue is near a transition point, and treating the boundary explicitly (rather than as a special case of normal operation) lets it batch operations more aggressively.

## Practical implications

- **Don't use a CPU queue on the GPU.** Even a "lock-free" CPU queue will lose to a blocking array queue at GPU thread counts.
- **Warp-centric is non-negotiable.** Any queue design that doesn't aggregate at warp granularity is starting at a 40× disadvantage.
- **The throughput ceiling is much lower** than CPU queues in absolute terms, but per-thread throughput is competitive given the parallelism.

## See also

- [[bacq]] — current state of the art
- [[mpmc-queue]] — CPU-side counterpart
- [[lcrq]] — the design that ported reasonably well to GPU
- [[the-fastest-queue]] — full hierarchy
