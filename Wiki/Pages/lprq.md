---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# LPRQ (Linked Portable Ring Queue)

LPRQ (Romanov & Koval, **PPoPP 2023**) is a direct portability fix for [[lcrq|LCRQ]]: it modifies the original algorithm to drop the 128-bit CAS while preserving identical performance on x86. It outperforms previous portable bounded MPMC queues by **up to 1.6×** and has been ported to Java, Kotlin, and Go.

## Why two portable LCRQs exist

[[scq|SCQ]] (DISC 2019) and LPRQ (PPoPP 2023) are independent solutions to the same problem — running LCRQ-class algorithms on architectures without `CMPXCHG16B`. They take different routes:

| | SCQ | LPRQ |
|---|---|---|
| Approach | Redesign around the threshold mechanism | Modify LCRQ directly |
| Memory | ~half of LCRQ (denser slot packing) | Comparable to LCRQ |
| Algorithm fidelity | Different from LCRQ | Closely tracks LCRQ |
| Cross-arch wins | Wins outright on PowerPC | Matches LCRQ on x86, beats portable predecessors by 1.6× |

In practice, SCQ wins when memory footprint matters and LPRQ wins when matching LCRQ's exact structure (e.g., when porting an existing LCRQ implementation) is preferable.

## How LPRQ avoids CAS2

The original LCRQ uses 128-bit CAS to atomically swap a `<value, index, safe>` triple. LPRQ splits this into a sequence of single-width operations on separate words, with carefully ordered visibility so the slot-level state machine remains linearizable. The cost is a slight increase in algorithmic complexity; the gain is portability to every architecture with single-width CAS — which is every architecture worth caring about.

## Performance

On the standard MPMC benchmarks:

- **Matches LCRQ** on x86 across the full thread-count range
- **Outperforms FAAArrayQueue and CRTurnQueue** by ~1.6× on ARM and PowerPC
- Available in production-quality form for Java (the paper's reference), Kotlin, and Go

## See also

- [[lcrq]] — the original
- [[scq]] — the alternative portable solution
- [[wcq]] — wait-free queue built on similar ideas
- [[the-fastest-queue]] — full hierarchy
