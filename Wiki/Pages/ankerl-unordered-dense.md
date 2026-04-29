---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# ankerl::unordered_dense

Martin Ankerl's single-header C++ hash map and the **uncontested iteration champion**. It stores all key-value pairs in a contiguous `std::vector` rather than the sparse slot array of [[swiss-table]] designs, making iteration as fast as scanning an array — 2.55× faster than [[abseil-flat-hash-map]] in Jackson Allan's benchmark. Robin Hood probing with backward-shift deletion sidesteps tombstone degradation entirely.

## The dense storage trick

A separate index table holds slot metadata; the actual entries live packed in a `std::vector`. Iteration walks the vector linearly with full hardware prefetching. Erase is a swap-and-pop that keeps the vector contiguous. The cost: insertions are slightly more expensive (the index update plus the vector append, 1.78× the leader on integer insert), and lookups touch one extra cache line (1.15× on lookup-hit — still very competitive).

## When it wins

- **Iteration-bound workloads** — analytics, snapshotting, walking a map repeatedly. Nothing else comes close.
- **Small-to-medium maps** (200–2,000 entries) where dense layout fits in L1/L2.
- **Single-header drop-in** — simplest possible integration. No Boost, no Abseil dependency.
- **Sustained insert/erase cycling** — Robin Hood backward-shift means no tombstone rot.

## When to pick something else

For pure single-threaded throughput on insert/erase-heavy workloads with little iteration, [[boost-unordered-flat-map]] wins. For lowest memory footprint, [[folly-f14]]. For concurrent access, see [[parlayhash]] / [[papaya]] / [[dashmap]].

## The broader landscape

ankerl::unordered_dense sits in the Jackson Allan benchmark alongside the Swiss Table family. Its dense-layout idea is conceptually closer to [[ecs-pattern|ECS archetype storage]] or [[soa-vs-aos|SoA layouts]] — pay a small lookup cost to make the dominant access pattern (sequential traversal) cache-perfect. See [[fastest-hash-map-2025]] for full context.
