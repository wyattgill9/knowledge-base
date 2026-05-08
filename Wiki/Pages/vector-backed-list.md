---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Vector-Backed Linked List

A linked list whose nodes live inside a single `std::vector` (or equivalent contiguous buffer), with index-based "pointers" instead of memory addresses. The closest a linked list gets to true array performance — at the cost of slightly more bookkeeping for free-slot management. The headline result: with explicit compaction to enforce sequential ordering, traversal hits the [[fastest-linked-lists|125× ceiling]] over naive heap-allocated lists.

## jsl::vector_list

Stores nodes inside a `std::vector` with `next`/`prev` as 32-bit indices rather than 64-bit pointers. A `compact()` operation rearranges nodes so logical order matches physical order, restoring sequential memory access:

- **7.8× faster traversal** than `std::forward_list` for 128M integers (with compaction).
- **5.25× faster traversal** even without compaction (just the vector layout).
- **3.2× faster insertion** than `std::forward_list`.

The compaction step is the explicit lever for the **125× memory-layout speedup** Johnny's Software Lab benchmark identified. `compact()` is O(n) but rare; amortized across millions of traversals between compactions, the cost is invisible.

## Rust: orx-linked-list

Claims **25× faster iteration** than `std::collections::LinkedList` through fragment-based contiguous storage — splitting the list into chunks of contiguous nodes that can be iterated as arrays. Rust's `LinkedList` is widely understood to be a misfeature in the standard library (its documentation now warns against using it); `orx-linked-list` is one of several community crates filling the gap.

## Java: GlueList

An unrolled list (each "node" holds an array of elements) hosted on top of an `ArrayList`-style backing buffer. Adds 1M elements in **39.2 ms** vs. `LinkedList`'s 174.8 ms — and beats `ArrayList`'s own 76.4 ms because the unrolled blocks avoid the periodic O(n) resize cost.

## The deeper point

Vector-backed lists make the [[fastest-linked-lists|"linked lists are obsolete"]] argument concrete. The structure walks like a linked list, quacks like a linked list, and benchmarks like an array. The 2024 "RIP Linked List" paper's **ArrayBlock** is the limit case: a block-based array structure that outperforms every linked-list variant *even on benchmarks designed to favor linked lists*. The conclusion the paper reaches — "it is very difficult to find much interest in using linked lists in real applications" — is increasingly hard to argue with.

## Where it sits

In the [[fastest-linked-lists|hierarchy]], vector-backed lists are the **traversal-throughput maximum** among linked-list-shaped structures. They lose to [[intrusive-list|intrusive lists]] when zero-allocation matters and to [[plf-list|plf::list]] when API compatibility with `std::list` matters. They beat both for read-heavy workloads where the list is mutated rarely and traversed often.

The same "use indices, store contiguously" pattern appears in [[csr-graph|CSR graph]] representations, [[ecs-pattern|ECS]] archetypes, and Bevy's component storage — the linked-list version is just the lowest-rank instance of the same idea.
