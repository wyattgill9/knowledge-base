---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# B-Tree

For ordered containers, the debate is settled: B-trees outperform red-black trees by 3–18x at scale due to cache efficiency. A `std::set<int32_t>` (red-black tree) consumes ~40 bytes per element with three pointers and a color bit per node, causing pointer-chasing cache misses on every comparison. B-trees pack dozens of keys into cache-line-sized nodes, slashing tree height by ~6x.

## The extreme end: Algorithmica S+ Tree

The **Algorithmica S+ Tree** (Sergey Slotin) achieves the most extreme results: 18x faster than `std::set` and 7x faster than `absl::btree` for `lower_bound` on integer keys. It uses branchless [[simd-programming|SIMD]] binary search within each node and compile-time constant height optimization in ~150 lines of C++. **frozenca::BTree** similarly uses SIMD node searching, benchmarking insert at 175ms vs 665ms for `std::set` on 1M int64 keys.

## Production implementations

Google's `absl::btree_map` provides a more production-ready option with ~62 children per node for `int32_t` and 4.3–5.1 bytes/value versus 40 bytes for `std::set`. It is the correct default for ordered containers in C++.

For databases, **LMDB** (Howard Chu) uses B+ trees with memory-mapped files and dominates read benchmarks against LevelDB, RocksDB, WiredTiger, and BerkeleyDB. The enhanced fork **libmdbx** claims 3x faster than BoltDB and up to 100,000x in extreme cases.

## Relationship to other tree structures

The [[adaptive-radix-tree]] outperforms B-trees for main-memory database lookups with variable-length keys, using four adaptive node sizes and SIMD-optimized search. [[learned-indexes]] challenge B-trees for static read-heavy workloads, delivering 2–3x faster lookups with smaller index sizes. [[cache-oblivious-structures|Cache-oblivious B-trees]] using van Emde Boas layout achieve optimal memory transfers without knowing cache parameters.

## Why B-trees win

The answer is always cache lines. A red-black tree node holds one key and three pointers — accessing any child requires a random memory access. A B-tree node holds dozens of keys contiguously, and searching within a node hits the same cache line(s). Modern B-trees add SIMD node search and branchless comparisons to further exploit hardware. This is the same principle that makes [[soa-vs-aos|SoA beat AoS]] and [[csr-graph|CSR beat adjacency lists]].
