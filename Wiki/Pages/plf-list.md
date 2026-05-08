---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# plf::list

Matt Bentley's drop-in `std::list` replacement, the **fastest general-purpose single-threaded linked list in C++**. Benchmarks on Haswell with GCC 8.1:

| Operation | Speedup vs `std::list` |
|-----------|------------------------|
| Insertion | 333% |
| Erasure | 81% |
| Iteration | 16% |
| Sort | 77% |
| `remove` / `remove_if` | 91% |
| `clear` | **6,500%** (1,147,900% for trivially-destructible types) |

## What makes it fast

Three reinforcing design choices:

1. **Unrolled block storage.** Nodes live in memory blocks of up to 2,048 elements rather than individual heap allocations — slashing allocator overhead, improving cache locality, and giving bulk operations large contiguous regions to scan. The `clear` speedup comes almost entirely from this: dropping a few large blocks vs. freeing millions of individual nodes.
2. **Spatial-aware reinsertion.** When inserting a new element, `plf::list` searches for the closest free slot to the insertion point so logically adjacent elements stay physically adjacent. This keeps cache locality from degrading over time as the list mutates.
3. **Block-linear bulk operations.** `reverse`, `clear`, and `sort` iterate over memory blocks linearly rather than following the linked-list `next` pointers — impossible with `std::list`'s scattered per-node allocation.

## Why std::list cannot match this

The C++ standard requires `std::list::splice` to be **O(1) for arbitrary sub-ranges**. To honor that, implementations cannot cluster nodes into shared blocks — partial-list splicing would either invalidate pointers or require O(n) work. `plf::list` deliberately gives up standard-compliant splice semantics to unlock block allocation.

This is a clean example of the C++ standard library's compatibility constraints costing measurable performance, and why third-party drop-ins (`plf::list`, `plf::colony`, [[absl-inlinedvector|abseil's containers]], [[boost-unordered-flat-map]]) consistently beat the standard equivalents for hot paths.

## Where it sits

In the [[fastest-linked-lists|fastest linked-list hierarchy]], `plf::list` is the single-threaded general-purpose leader. For raw traversal, [[vector-backed-list|vector-backed lists]] with explicit compaction are faster (up to 125× over naive layouts). For zero-allocation systems use, the [[intrusive-list|Linux kernel `list.h`]] pattern is the standard.

The same block-allocation principle drives [[plf-colony|plf::colony]] (Bentley's hash-set-shaped variant) and is the deeper reason linked lists *work* when properly designed: contiguous blocks tame the cache penalty that gives `std::list` its bad reputation.
