---
tags:
  - concurrency
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Hazard Pointers

A safe-memory-reclamation scheme for lock-free data structures: each thread publishes the pointers it is currently dereferencing in a small per-thread "hazard" array; reclamation skips any node whose address appears in some thread's hazard list. **Standardized in C++26.**

Maged Michael (2002, 2004) introduced hazard pointers as a robust alternative to garbage-collected reclamation in concurrent algorithms. The defining property is **bounded memory**: each thread holds at most a constant number of hazards, so the worst-case unreclaimed-but-retired set size is bounded by O(threads × hazards-per-thread). This makes hazard pointers the right primitive when the system absolutely cannot tolerate unbounded retention — kernel code, real-time systems, anything sharing memory with a watchdog.

## The trade with EBR

[[crossbeam-epoch|Epoch-based reclamation (EBR)]] is faster on the common path — it amortizes reclamation across batches of operations and adds essentially zero overhead per access. Its weakness is the *grace period*: if any thread stalls inside an epoch (descheduled, page-faulted, blocked on I/O), reclamation cannot proceed and retired memory accumulates. Under adversarial scheduling, EBR retention is unbounded.

Hazard pointers invert the trade-off:

| Property | EBR | Hazard pointers |
|----------|-----|-----------------|
| Common-path cost | ~zero | One store + fence per access |
| Worst-case retention | Unbounded | Bounded |
| Stalled-thread tolerance | Poor | Good |
| Standardization | None (per-runtime) | **C++26** |

The per-access overhead — typically a single relaxed store followed by a load fence — is small but real. For read-heavy workloads where every cache line matters (think [[swiss-table|Swiss Table]] probes), the overhead matters; for write-heavy or contention-heavy workloads it disappears in the noise.

## C++26 standardization

C++26 adds `std::hazard_pointer` and `std::hazard_pointer_obj_base<T>` as standard library facilities. The same ISO C++ paper trail that produced [[trivial-relocatability|P1144]] and `std::flat_map` (Swiss-Table-shaped) also produced hazard pointers — the standardization wave is reaching primitives that were previously confined to per-runtime libraries (folly, JCTools, crossbeam).

## Crystalline: bounded *and* wait-free

For decades, the conventional wisdom was that you had to choose between EBR's speed and hazard pointers' bounded memory. **Crystalline** ([[crystalline-reclamation]], Nikolaev & Ravindran, PLDI 2024) achieves wait-free reclamation with bounded memory simultaneously — a result previously thought impossible. Crystalline is the new frontier; hazard pointers remain the standardized, well-understood baseline.

## Where it sits

In the [[fastest-linked-lists|linked-list]] and [[the-fastest-queue|queue]] landscapes, hazard pointers are the reclamation choice for **systems-grade** concurrent code where worst-case bounds matter — kernel-adjacent work, persistent memory ([[direct-io]]-style storage layers), real-time. For Rust, [[crossbeam-epoch]] is the EBR analogue and the dominant choice for general-purpose concurrent data structures. The two are complements, not substitutes.

The wider memory reclamation landscape now spans EBR, hazard pointers, [[rcu|RCU]], reference counting, and Crystalline — picking the right one is often more impactful than picking a different concurrent algorithm.
