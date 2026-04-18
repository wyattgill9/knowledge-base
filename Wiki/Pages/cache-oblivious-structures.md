---
tags:
  - data-structures
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Cache-Oblivious Data Structures

Data structures that optimize for every cache level simultaneously without knowing cache parameters (line size, cache size, associativity). They achieve optimal memory transfer complexity for any memory hierarchy — from L1 through L3 to disk — using a single implementation.

## The van Emde Boas layout

The foundational technique: recursively split a tree and lay subtrees contiguously in memory. A balanced binary tree of height h is split at height h/2, placing the top half first, then each bottom subtree contiguously. Applied recursively, this achieves optimal O(log_B N) memory transfers for search, where B is the (unknown) cache line size.

This layout underpins **cache-oblivious B-trees** (Bender, Demaine, Farach-Colton) — B-tree performance without needing to tune the node size to the cache line.

## Cache-oblivious lookahead arrays (COLAs)

COLAs showed **790x faster** than [[b-tree|B-trees]] for large random insertions. They use a series of sorted arrays of exponentially increasing size (1, 2, 4, 8, ...) with a merge-based insertion strategy. Insertions are amortized O((log² N)/B) memory transfers — dramatically better than B-trees for write-heavy workloads.

## Practical significance

Cache-oblivious structures are most valuable when:
- The data structure must perform well across different machines with different cache hierarchies.
- The working set size varies across the cache hierarchy (fits in L1 sometimes, spills to L3 other times).
- You cannot or do not want to tune parameters to specific hardware.

They complement [[learned-indexes]] (which adapt to data distribution) and [[succinct-data-structures]] (which minimize space) as advanced alternatives to classical structures.
