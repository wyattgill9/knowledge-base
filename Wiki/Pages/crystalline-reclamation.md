---
tags:
  - concurrency
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Crystalline

Nikolaev & Ravindran, PLDI 2024. A safe-memory-reclamation scheme that achieves **wait-free reclamation with bounded memory** simultaneously — a combination previously considered impossible. The conventional trade-off was [[crossbeam-epoch|EBR]] (fast, unbounded under stalled threads) versus [[hazard-pointers]] (bounded, per-access overhead); Crystalline gets both halves.

## The result

Wait-freedom for memory reclamation means **every thread makes progress in a bounded number of steps regardless of other threads' state** — no thread can stall reclamation by being descheduled, page-faulted, or blocked. Bounded memory means **the worst-case retired-but-not-reclaimed set is constant in the thread count**, regardless of how long any thread takes per operation.

EBR sacrifices boundedness — a stalled thread holds an epoch open and retired memory accumulates indefinitely. Hazard pointers sacrifice common-path speed — every access pays a store + fence. Crystalline avoids both sacrifices using a more sophisticated tracking discipline that the paper proves correct against the standard memory model.

## Why it matters

Memory reclamation is **the dominant cost factor for lock-free data structures**, often beating the algorithmic cost of the structure itself. The [[bw-tree]]'s loss to [[art-olc|ART-OLC]] has roughly the same flavor — reclamation traffic on shared cache lines drowns the algorithmic win. Crystalline shifts the design space: structures that previously had to choose between fast-but-unbounded EBR and bounded-but-slow hazard pointers can now have both.

For the [[fastest-linked-lists|fastest linked lists]] this matters most acutely on long lock-free traversal paths where hazard pointer's per-access overhead added up. Combined with [[harris-linked-list|Harris-Träff-Pöter]] marking, Crystalline-managed reclamation removes the last major source of overhead from lock-free ordered list operations.

## Where it sits

Crystalline is the **2024 frontier** of memory reclamation. Adoption in production runtimes (folly, crossbeam, JCTools) is the next milestone — the algorithm is published; the integration work follows. For now, [[crossbeam-epoch|EBR]] remains the Rust ecosystem default and [[hazard-pointers]] are the C++26 standardized choice; Crystalline is what you reach for when both prior options are visibly costing you.

The broader pattern across the 2024–2026 wave: theoretical impossibility results often turn out to assume more than necessary. Crystalline joins [[elastic-hashing]] (which disproved Yao's 1985 conjecture on probe complexity) as a recent example of a "settled" lower bound being broken.
