---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# d-ary Heap

The fastest practical heap for most workloads. A landmark 2014 study by Larkin, Sen, and Tarjan definitively showed that theoretical optimality does not predict practical heap performance — and d-ary heaps with d=4 consistently win.

## Why d=4

A 4-ary heap stores elements in a flat array (excellent spatial locality) with 4 children per node instead of 2. The wider fanout reduces tree height and cache misses, delivering **17–30% speedup over binary heaps** when the heap exceeds L1 cache. The sweet spot is d=4: enough fanout to reduce height meaningfully, not so much that per-node comparison costs dominate.

## The Fibonacci heap trap

Fibonacci heaps have O(1) amortized decrease-key, making them theoretically optimal for Dijkstra's and Prim's algorithms. In practice, they are consistently slower than simpler alternatives due to 4 pointers per node, poor cache locality, and high constant factors. **Brodal queues** achieve optimal worst-case bounds but are, in Brodal's own words, "quite complicated" and "not applicable in practice" — only one known implementation exists (a Scala functional version).

## When you need decrease-key

For algorithms requiring decrease-key (Dijkstra's, Prim's), **pairing heaps** are the fastest pointer-based option. They are simpler than Fibonacci heaps with only a two-pass merge on extract-min, and "almost always faster than other pointer-based heaps including Fibonacci heaps" per the Larkin/Sen/Tarjan study.

## For very large datasets

When data exceeds cache, **sequence heaps** (Sanders, 1999) based on k-way merging are at least 2x faster than optimized binary heaps and 4-ary heaps. These exploit cache-oblivious merge patterns to minimize memory transfers.

## Practical guidance

Use a 4-ary heap for simple priority queue workloads. Use a pairing heap when decrease-key is needed. Don't use Fibonacci heaps — the theory doesn't translate. For very large heaps exceeding cache, investigate sequence heaps or [[cache-oblivious-structures|cache-oblivious]] alternatives.
