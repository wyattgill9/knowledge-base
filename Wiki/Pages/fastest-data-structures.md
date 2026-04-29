---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# Fastest Data Structures in Computer Science

A benchmarked survey of the fastest known implementations across ten major data structure categories, drawing on real-world results from 2020–2026. The single biggest performance lever in modern data structure design is not algorithmic complexity — it is cache locality. Across every category, implementations that minimize cache misses and exploit [[simd-programming|SIMD instructions]] dominate benchmarks, often by 5–20x over theoretically equivalent alternatives.

## The winners, category by category

| Category | Champion | Key advantage |
|----------|----------|---------------|
| Hash maps | [[swiss-table]] variants — [[boost-unordered-flat-map]] leads, [[abseil-flat-hash-map]] close second, [[hashbrown]] in Rust | SIMD-parallel metadata probing, 16 slots in ~3 instructions |
| Ordered containers | [[b-tree]] (not red-black trees) | 3–18x faster due to cache-line-sized nodes, 6x lower tree height |
| Tries & strings | [[adaptive-radix-tree]] | Four adaptive node sizes with SIMD search, path compression |
| Heaps | [[d-ary-heap]] (d=4) | Wider fanout reduces cache misses; 17–30% faster than binary heaps |
| Sequential sort | [[pdqsort]] | Linear on presorted data, heapsort worst-case guarantee, branchless variant |
| Parallel sort | [[ips4o]] | 1.5x faster than next sequential, 3x faster than next parallel competitor |
| Stable sort | [[driftsort]] (Rust) | Up to 4x faster than previous Rust stable sort |
| Lock-free queues | [[lmax-disruptor]] | 52 ns/hop — 630x faster than `ArrayBlockingQueue` |
| Read-dominated concurrency | [[rcu]] | Zero read-side overhead — no locks, no atomics, no barriers |
| Set membership | [[binary-fuse-filter]] | 13% of information-theoretic minimum, 2x faster construction than Xor |
| Graph analytics | [[csr-graph]] | Contiguous edge arrays, maximal cache locality, 40–250x faster than NetworkX |
| Static lookups | [[learned-indexes]] | 2–3x faster than B-trees with smaller index sizes |

## Three principles that make data structures fast

**SIMD-parallel metadata scanning.** Pioneered by [[swiss-table]] and now used in hash maps, tries ([[adaptive-radix-tree]] Node16), Bloom filters, and [[b-tree]] node search. Transforms sequential comparisons into batch operations — checking 16 slots in 3 instructions.

**Contiguous memory layouts.** Flat arrays, [[soa-vs-aos|SoA]], packed nodes. These exploit hardware prefetchers and cache lines. B-trees beat red-black trees by 18x. Flat arrays outperform linked lists by 10–100x. The [[ecs-pattern]] operationalizes this at scale in game engines.

**Branch elimination.** Branchless partitioning in [[pdqsort]]/[[driftsort]], branchless SIMD search in Algorithmica's B-tree, branchless merge loops. Modern CPUs pay enormous costs for branch mispredictions — eliminating them is consistently worth 20–50% performance.

## The practical default stack

Use [[boost-unordered-flat-map]] or [[abseil-flat-hash-map]] for C++ hash maps ([[hashbrown]] in Rust). Note: in many workloads the *hash function* (foldhash, rapidhash) matters more than the table — see [[fastest-hash-map-2025]]. Use [[b-tree]] variants for ordered containers. Use [[d-ary-heap]] for priority queues. Use [[ips4o]] for parallel sorting. Use [[binary-fuse-filter]] for set membership. Use [[csr-graph]] for graph analytics. For concurrent hash maps at scale, [[parlayhash]] hits 1,130 Mops at 128 threads. For other concurrent workloads, the [[lmax-disruptor]] pattern and [[rcu]] remain hard to beat. The most exciting frontier — [[learned-indexes]] — is already delivering production wins and may fundamentally reshape database indexing.

## Allocation and memory layout

[[mimalloc]] consistently outperforms jemalloc, tcmalloc, and system malloc — 13% faster on the Lean compiler, 15% lower P99 latency for small allocations. Arena/bump allocators like [[bumpalo]] deliver 2–5x speedup for batch allocations. The [[soa-vs-aos]] pattern yields 1.4–12.7x speedup depending on struct size and access pattern. [[simdjson]] demonstrates the ceiling of SIMD-optimized design: 4x faster than RapidJSON, 25x faster than JSON for Modern C++.

## Emerging frontiers

[[learned-indexes]] replace B-tree branching with piecewise-linear CDF approximation for O(log log N) lookup. [[cache-oblivious-structures]] optimize for every cache level simultaneously without knowing cache parameters. [[succinct-data-structures]] trade CPU cycles for near-information-theoretic space efficiency. [[elastic-hashing]] (Jan 2025) disproved Yao's 1985 conjecture on probe complexity, suggesting hash table design is not yet exhausted. These are not just academic — RadixSpline is in RocksDB, PGM-Index has production deployments, and SDSL-lite implements 40 research publications in usable C++ templates.
