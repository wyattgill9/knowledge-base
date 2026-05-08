---
tags:
  - data-structures
  - type-theory
  - performance
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Persistent Functional List

A singly-linked `cons` list with **structural sharing**: every "modification" returns a new list head while sharing all unchanged tails with the prior version. The backbone of Haskell, Clojure, Scala, OCaml, and ML collections. Despite the [[fastest-linked-lists|125× cache penalty]] of pointer-chasing on modern hardware, persistent lists remain genuinely competitive in their narrow domain because they buy properties contiguous structures fundamentally cannot offer.

## What you get

- **O(1) prepend.** A new list head pointing at the existing list — one cell allocated, the rest shared.
- **Lock-free thread safety.** Old versions are immutable; readers cannot conflict with writers because writers never mutate. No locks, no atomics, no [[crossbeam-epoch|reclamation scheme]] beyond garbage collection.
- **Time-travel.** Every prior version is still reachable as long as a reference to its head exists. Undo, transactional snapshots, and persistent data structures fall out for free.
- **Structural sharing.** Two lists differing in their first three elements but agreeing on the rest share the suffix — memory cost is O(diff), not O(n + m).

These properties compound: an immutable persistent list is exactly the right data structure for concurrent message queues in actor systems, snapshot isolation in software transactional memory, and the AST representations that make purely functional compilers tractable.

## What you give up

Cache locality. A Haskell `[Int]` is a heap-allocated chain of `(Int, ListPointer)` cells, which is exactly the worst case in the [[fastest-linked-lists|125× layout benchmark]]. Traversal pays a cache miss per element. For arithmetic-on-arrays workloads, persistent lists are a poor choice and Haskell programmers reach for `Data.Vector` or unboxed arrays.

Indexing is O(n), not O(1). Anything that needs random access uses `Data.Sequence` (a finger-tree), `Data.IntMap` (a [[champ|HAMT]]-style trie), or one of the persistent vector designs. The persistent list is the *primitive* in the functional toolbox, not the universal container.

## The structural-sharing connection

The same structural-sharing principle scales up to richer immutable structures: **HAMT** (Hash Array Mapped Tries) and the [[champ]] (Compressed HAMT) variant power Clojure's persistent maps and sets, achieving 10–100% improvements over HAMT alone. Persistent vectors with 32-way branching (Clojure, Scala) match `ArrayList` indexing performance to within a small constant while preserving immutability. The persistent list is the rank-1 case of this entire design philosophy.

## Where it sits

In the [[fastest-linked-lists|hierarchy]], persistent functional lists occupy the **persistence-mandatory tier**. They lose to every cache-friendly variant on raw throughput but win uniquely when:

- The workload demands snapshot isolation or time-travel.
- The collection is read by many threads concurrently with writers (immutability provides lock-free read semantics).
- Memory cost must scale with diff size, not version count.

For mutable concurrent linked lists choose [[harris-linked-list|Harris-Träff-Pöter]]. For mutable single-threaded use choose [[plf-list]] or [[vector-backed-list|vector-backed lists]]. For persistent ordered or indexed access, look beyond the linked list to [[champ]], [[fst-finite-state-transducer|FSTs]], and finger trees. The persistent linked list is small, simple, and uniquely suited to one shape of problem.
