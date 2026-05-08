---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Unrolled Linked List

A linked list where each node holds a **small array of m elements** sized to fill one or more cache lines, rather than a single element. Cache misses drop by a factor of m; indexing requires only n/m + 1 misses versus n for a standard list, approaching the optimal n/B bound where B is the cache-line capacity. Research consistently shows **2–4× traversal speedups** and **~60% fewer cache misses** versus standard linked lists.

## The structural argument

A standard linked list with k bytes per element wastes (cache_line_size − k) bytes per node — roughly 56 bytes on a 64-byte line for a typical 8-byte payload + 16 bytes of pointers. An unrolled node fills the cache line with payload, paying the pointer cost once per m elements rather than once per element. The cleaner asymptotic statement: indexing complexity drops from O(n) cache misses to O(n/m), and m is chosen to make the per-node array fill an integer number of cache lines.

This is the same locality argument behind [[b-tree|B-trees]], [[adaptive-radix-tree|ART]] node packing, and [[ecs-pattern|ECS]] archetype storage. Unrolled linked lists are the linked-list-shaped instance of that pattern.

## Concurrent variants

The **concurrent unrolled linked list with lazy synchronization** (Platz, Mittal, Venkatesan, JPDC 2020) achieves **300% higher throughput** than other concurrent list-based sets, including Braginsky-Petrank's locality-conscious lists. Wait-free reads and single-node locking for most writes make it practical for real workloads.

**DULL** (Detectable Unrolled Lock-based Linked List, OPODIS 2024) extends the unrolled design to persistent memory, delivering several-fold faster performance than competitors for update-intensive workloads while guaranteeing crash consistency.

## Cache-conscious STL lists

Frias, Petit, and Roura (2006) formalized the unrolled approach as a doubly-linked list of *buckets*, each holding Θ(B) elements tuned to cache-line size. Their **2-level contiguous list** variant achieves traversal performance **independent of total list size** when buckets are properly sized — logically adjacent elements always share cache lines.

## Production examples

Java's **GlueList** is an unrolled linked list that adds 1 million elements in **39.2 ms** versus `LinkedList`'s 174.8 ms (4.5× faster) and even beats `ArrayList`'s 76.4 ms. [[plf-list|plf::list]] uses unrolled storage in 2,048-element blocks as one of its three core mechanisms, contributing to the 6,500% `clear` speedup over `std::list`.

The [[slab-list]] design adapts the unrolled idea to GPU warps, where each "slab" is unrolled to fit a 32-thread warp width.

## Where it sits

Unrolled linked lists are the **best general cache-to-flexibility trade-off** in the [[fastest-linked-lists|linked-list hierarchy]]. They preserve O(1) insertion at known positions while paying near-array traversal cost. For maximum traversal throughput, [[vector-backed-list|vector-backed lists]] with explicit compaction go further (125× over naive layouts). For minimum overhead, [[intrusive-list|intrusive lists]] eliminate node allocation entirely. The unrolled approach is the right default when neither of those constraints applies.
