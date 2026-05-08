---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Fastest Data Structures in Computer Science

A benchmarked survey of the fastest known implementations across ten major data structure categories, drawing on real-world results from 2020–2026. The single biggest performance lever in modern data structure design is not algorithmic complexity — it is cache locality. Across every category, implementations that minimize cache misses and exploit [[simd-programming|SIMD instructions]] dominate benchmarks, often by 5–20x over theoretically equivalent alternatives.

## The winners, category by category

| Category | Champion | Key advantage |
|----------|----------|---------------|
| Dynamic arrays | C `realloc` + [[mimalloc]]/[[jemalloc]]; [[folly-fbvector]] in C++ | [[realloc-mremap\|mremap]] enables O(1) growth without copying; [[fastest-dynamic-arrays]] |
| Hash maps | [[swiss-table]] variants — [[boost-unordered-flat-map]] leads, [[abseil-flat-hash-map]] close second, [[hashbrown]] in Rust | SIMD-parallel metadata probing, 16 slots in ~3 instructions |
| Ordered containers | [[bs-tree]] / [[algorithmica-s-tree]] (SIMD [[b-tree]]) | 7–18× over `std::map`; 16 keys/node in 2 AVX-512 instructions; [[fastest-ordered-maps]] |
| Concurrent ordered maps | [[art-olc]] / [[bp-tree]] / [[masstree]] | OLC readers never write shared cache lines; lock-free [[bw-tree]] is 1.5–4.5× *slower* |
| Linked lists (single-threaded) | [[plf-list]] / [[vector-backed-list]] / [[intrusive-list]] | Block allocation, contiguous index linking, embedded pointers; up to **125× over naive layouts**; [[fastest-linked-lists]] |
| Lock-free ordered linked lists | [[harris-linked-list\|Harris-Träff-Pöter]] / VBL | `fetch_or` marking removes the CAS-retry retraversal cost |
| Tries & strings | [[adaptive-radix-tree]], [[hot-trie]] | Four adaptive node sizes with SIMD search; HOT varies bits per node by distribution |
| Heaps | [[d-ary-heap]] (d=4) | Wider fanout reduces cache misses; 17–30% faster than binary heaps |
| Sequential sort | [[pdqsort]] | Linear on presorted data, heapsort worst-case guarantee, branchless variant |
| Parallel sort | [[ips4o]] | 1.5x faster than next sequential, 3x faster than next parallel competitor |
| Stable sort | [[driftsort]] (Rust) | Up to 4x faster than previous Rust stable sort |
| MPMC queues | [[lcrq]] + [[aggregating-funnels]] | FAA-on-a-ring; 2.5× over plain LCRQ at high thread counts; [[the-fastest-queue]] |
| SPSC queues | [[ring-buffer]] (acquire/release + [[false-sharing\|128-byte padding]]) | 300–530M ops/s; [[rtrb]] hits 520M+ on Apple M4 |
| Bounded latency pipelines | [[lmax-disruptor]] | 52 ns/hop — 630× lower mean latency than `ArrayBlockingQueue` |
| Read-dominated concurrency | [[rcu]] | Zero read-side overhead — no locks, no atomics, no barriers |
| Set membership | [[binary-fuse-filter]] | 13% of information-theoretic minimum, 2x faster construction than Xor |
| Graph analytics | [[csr-graph]] | Contiguous edge arrays, maximal cache locality, 40–250x faster than NetworkX |
| Static lookups | [[learned-indexes]] | 2–3x faster than B-trees with smaller index sizes |

## Three principles that make data structures fast

**SIMD-parallel metadata scanning.** Pioneered by [[swiss-table]] and now used in hash maps, tries ([[adaptive-radix-tree]] Node16), Bloom filters, and [[b-tree]] node search. Transforms sequential comparisons into batch operations — checking 16 slots in 3 instructions.

**Contiguous memory layouts.** Flat arrays, [[soa-vs-aos|SoA]], packed nodes. These exploit hardware prefetchers and cache lines. B-trees beat red-black trees by 18x. Flat arrays outperform linked lists by 10–100x. The [[ecs-pattern]] operationalizes this at scale in game engines.

**Branch elimination.** Branchless partitioning in [[pdqsort]]/[[driftsort]], branchless SIMD search in Algorithmica's B-tree, branchless merge loops. Modern CPUs pay enormous costs for branch mispredictions — eliminating them is consistently worth 20–50% performance.

**[[faa-vs-cas|FAA over CAS]] at contention hotspots.** The principle that obsoleted the [[michael-scott-queue]] and produced [[lcrq|LCRQ]], [[scq|SCQ]], [[lprq|LPRQ]], and [[wcq|wCQ]]: CAS-retry melts down under contention while FAA always succeeds. At 144 threads: FAA ~40 ns vs CAS ~75 ns. Combined with the [[false-sharing|128-byte cache-line rule]], this is the dominant lever in concurrent data structure design.

**Optimistic readers that never write shared memory.** The same insight at the ordered-map layer: [[art-olc|ART-OLC]] and [[masstree]] beat the lock-free [[bw-tree]] by 1.5–4.5× because lock-freedom is not cache-coherence-freedom. Version-counter validation lets unbounded reader counts share clean cache lines. This unifies [[rcu]], FAA-over-CAS, and modern concurrent ordered maps into a single design discipline: avoid writes to shared cache lines on the hot path.

## The practical default stack

Use [[boost-unordered-flat-map]] or [[abseil-flat-hash-map]] for C++ hash maps ([[hashbrown]] in Rust). Note: in many workloads the *hash function* (foldhash, rapidhash) matters more than the table — see [[fastest-hash-map-2025]]. Use [[abseil-flat-hash-map|Abseil]] `btree_map` for production ordered containers; reach for [[bs-tree]] / [[algorithmica-s-tree]] / [[fb-plus-tree]] when squeezing the last 7–18× matters; reach for [[adaptive-radix-tree|ART]] / [[hot-trie|HOT]] for in-memory database indexes — see [[fastest-ordered-maps]]. Use [[d-ary-heap]] for priority queues. Use [[ips4o]] for parallel sorting. Use [[binary-fuse-filter]] for set membership. Use [[csr-graph]] for graph analytics. For concurrent hash maps at scale, [[parlayhash]] hits 1,130 Mops at 128 threads. For concurrent queues, [[lcrq|LCRQ]] + [[aggregating-funnels]] is the strict-FIFO MPMC ceiling; [[lmax-disruptor]] still wins for bounded pipeline tail latency; for read-dominated workloads [[rcu]] remains hard to beat. The most exciting frontier — [[learned-indexes]] — is already delivering production wins and may fundamentally reshape database indexing.

## Allocation and memory layout

The single most underrated lever in data structure performance: **the allocator dominates the container** — switching glibc → [[mimalloc]]/[[jemalloc]]/[[tcmalloc]] swings benchmarks more than picking a "better" container does. [[fastest-dynamic-arrays|Dynamic-array benchmarks]] make this concrete: C, Rust, and C++ vectors all land within ~10% of each other once allocators are matched, but switching allocators alone moves results 2–3×. Arena/bump allocators like [[bumpalo]] deliver 2–5x speedup for batch allocations. The [[soa-vs-aos]] pattern yields 1.4–12.7x speedup depending on struct size and access pattern. [[simdjson]] demonstrates the ceiling of SIMD-optimized design: 4x faster than RapidJSON, 25x faster than JSON for Modern C++.

## Emerging frontiers

[[learned-indexes]] replace B-tree branching with piecewise-linear CDF approximation for O(log log N) lookup. [[cache-oblivious-structures]] optimize for every cache level simultaneously without knowing cache parameters. [[succinct-data-structures]] trade CPU cycles for near-information-theoretic space efficiency. [[elastic-hashing]] (Jan 2025) disproved Yao's 1985 conjecture on probe complexity, suggesting hash table design is not yet exhausted. These are not just academic — RadixSpline is in RocksDB, PGM-Index has production deployments, and SDSL-lite implements 40 research publications in usable C++ templates.
