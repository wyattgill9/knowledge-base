---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# Iceberg Hashing

A hash table design (JACM 2023) that simultaneously optimizes **space, cache efficiency, and stability** — the third property being unusually rare. Stability here means elements rarely move once inserted, which matters enormously for persistent memory and disk-backed structures where every relocation is an I/O.

## Why three-axis optimization is hard

Most hash maps optimize two of the three axes and give up the third:

- [[swiss-table]] designs optimize space and cache, sacrifice stability (elements move on rehash and Robin Hood reorganization).
- Linked-list separate chaining optimizes stability and (sort of) space, sacrifices cache.
- Cuckoo hashing optimizes cache and (at low load) space, but moves elements aggressively to maintain its invariant.

Iceberg combines a small fast in-memory level with a larger overflow level, with an insertion policy designed to keep movement bounded. Most operations stay in the fast level; the large level absorbs overflow without forcing reorganization of already-placed elements.

## Where it fits

Iceberg's stability property makes it especially attractive for:

- **Persistent memory** — every element move is an expensive PMEM write.
- **Disk-backed indexes** — moves cause page rewrites.
- **Memory-mapped tables** — moves invalidate user references and break zero-copy paths.

For pure in-memory workloads where movement is cheap, the [[swiss-table]] family ([[boost-unordered-flat-map]], [[abseil-flat-hash-map]]) still wins on raw throughput. Iceberg's tradeoff only pays off when movement carries an asymmetric cost.

## In the broader theoretical picture

Iceberg sits alongside [[elastic-hashing]] (2025) as evidence that the hash map design space is not exhausted. The Swiss Table convergence dominates the practical landscape, but specific structural properties — stability, very high load factor tolerance, optimal probe complexity — still drive new theoretical work. See [[fastest-hash-map-2025]] for context.
