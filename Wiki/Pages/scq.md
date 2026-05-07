---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# SCQ (Scalable Circular Queue)

SCQ (Nikolaev, **DISC 2019**) is the lock-free bounded MPMC ring buffer that finally made [[lcrq|LCRQ]]-class performance portable. It uses only single-width CAS and FAA — no `CMPXCHG16B` — so it runs on every architecture, including ARM, RISC-V, and PowerPC. It matches LCRQ throughput on x86, **wins outright on PowerPC**, and uses roughly **half the memory** of LCRQ.

## What SCQ replaced

LCRQ's reliance on 128-bit CAS confined it to x86 for six years. Several portable alternatives existed (FAAArrayQueue, CRTurnQueue), but they either gave up performance or were unbounded. SCQ is the first portable bounded MPMC queue to match LCRQ on x86 — and the first to actually beat it on architectures without CAS2.

## The threshold mechanism

The key technical contribution is the **threshold** field, which replaces LCRQ's per-slot `safe` bit. The threshold tracks how far behind the consumer can be without livelocking; when a slow producer is about to write into a slot the consumer has given up on, the threshold tells it to bail. This is what enables SCQ to:

- Pack slots without per-slot padding (saving ~half the memory vs LCRQ)
- Use only single-width CAS (the slot-level synchronization fits in 64 bits)
- Avoid the livelocks that plague simpler `<head, tail>`-only ring designs

## Memory footprint

LCRQ's per-slot 64-byte cache-line padding pushes its memory consumption to ~400 MB under load on a typical configuration. SCQ packs slots more densely while preserving the no-[[false-sharing|false-sharing]] guarantee on the contended `head`/`tail` indices, dropping memory roughly **2×** without losing throughput. On PowerPC the footprint advantage compounds with the absence of CAS2 overhead.

## Performance

Published numbers from the DISC 2019 paper and follow-up benchmarks:

- **Matches LCRQ on x86** across all thread counts tested
- **Outperforms LCRQ on PowerPC** (LCRQ falls back to expensive CAS2 emulation)
- **Outperforms FAAArrayQueue and previous portable bounded queues by ~2×**

## Successors

[[lprq|LPRQ]] (PPoPP 2023) is a different solution to the same portability problem — it modifies LCRQ directly rather than redesigning around the threshold mechanism. SCQ remains the best choice when memory footprint matters; LPRQ tends to win when matching LCRQ semantics exactly is required.

[[wcq|wCQ]] (SPAA 2022) layers wait-freedom on top of SCQ via fast-path-slow-path: SCQ runs the fast path, and only after MAX_PATIENCE failures does a thread cooperate via the slow path. The slow path almost never fires in practice.

## See also

- [[lcrq]] — the algorithm SCQ was designed to portably replicate
- [[lprq]] — the alternative portable LCRQ
- [[wcq]] — wait-free queue built on SCQ
- [[the-fastest-queue]] — full hierarchy
- [[faa-vs-cas]] — the underlying primitive choice
