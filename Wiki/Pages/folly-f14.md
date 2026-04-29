---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-29
---

# folly F14

Meta's hash map family. The headline number is memory: **24.7 bytes per element** for `F14ValueMap`, the lowest among high-performance hash maps, while still maintaining good lookup speed. F14 takes a different structural approach from [[swiss-table]] — 14-entry chunks co-locating metadata and data, with overflow counting tracing back to Amble-Knuth 1974 rather than tombstones.

## The variants

- **`F14ValueMap`** — values stored inline. Best for small value types.
- **`F14NodeMap`** — values stored via pointer. Stable references across rehash.
- **`F14VectorMap`** — values in a `std::vector`. Iteration-friendly, like [[ankerl-unordered-dense]].
- **`F14FastMap`** — alias for the variant Meta picks for your value size; usually `F14ValueMap` when `sizeof(V) < 24`.

## Why the chunked layout

Each chunk holds 14 entries plus metadata in a single cache-friendly block. SIMD probing operates within the chunk. Co-location means a cache miss to the metadata also pulls in the entries themselves — fewer total memory transactions on hits. The tradeoff is slightly more complex iteration and rehashing logic than flat Swiss Table designs.

Nathan Bronson reported F14 beating [[abseil-flat-hash-map|Abseil's]] Swiss Table on some of Google's own benchmarks. Headline-throughput leadership has since moved to [[boost-unordered-flat-map]], but F14 retains the memory-efficiency crown.

## Overflow counting, not tombstones

Instead of marking deleted slots with sentinels (which accumulate and rot probe chains — see [[abseil-flat-hash-map]]), F14 maintains a per-chunk count of how many overflows the chunk has experienced. Lookups that don't find a match can terminate when the count is zero. The technique predates Swiss Tables by decades; [[boost-unordered-flat-map]]'s overflow byte is a closely related modern reinvention.

## When to choose F14

- **Memory-constrained workloads** at scale — the 24.7 bytes/elt advantage compounds across millions of maps.
- **Small value types** (sizeof < 24) — `F14ValueMap` is the design's sweet spot.
- **Inside Meta's stack** — F14 is the production reference; folly tooling expects it.

For pure throughput, [[boost-unordered-flat-map]] is faster. For iteration, [[ankerl-unordered-dense]]. For concurrent loads, `folly::ConcurrentHashMap` is competitive but [[parlayhash]] dominates at scale (157 vs 1,130 Mops at 128 threads).
