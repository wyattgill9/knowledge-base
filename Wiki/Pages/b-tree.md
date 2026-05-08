---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# B-Tree

For ordered containers, the debate is settled: **SIMD B-trees outperform red-black trees by 7–18×** for lookups and 3–8× for inserts on modern hardware. A `std::set<int32_t>` (red-black tree) consumes ~40 bytes per element with three pointers and a color bit per node, causing pointer-chasing cache misses on every comparison. A B-tree with fanout 62 packs dozens of keys into cache-line-sized nodes, collapses tree height from ~20 levels to 3–4, and SIMD-scans each node in a single clock cycle. The era of `std::map`, Java `TreeMap`, and other red-black-tree-backed standard containers is decisively over for performance-critical code.

## Why B-trees win

A single DRAM cache miss costs **60–100 ns** (~150–300 cycles); an L1 hit costs ~1 ns. A red-black tree at 1M keys is ~20 levels deep and chases a pointer at every level — almost every comparison is a cache miss. A B-tree with fanout 62 collapses that to 3–4 levels and searches each node with one SIMD instruction. The compounding effect:

| Factor | `std::map` (red-black) | [[abseil-flat-hash-map\|Abseil]] `btree_map` | SIMD B-tree |
|--------|------------------------|----------------------------------------------|-------------|
| Tree height (1M int32) | ~20 | ~3–4 | ~3 |
| Bytes per element | 40 | ~5 | ~5.5 |
| Cache misses / lookup | ~20 | ~4 | ~3 |
| Intra-node search | scalar, 1 key | linear, ~30 keys | **SIMD, 8+ keys/cycle** |
| Branch mispredicts / lookup | ~10 | ~3–4 | **~0 (branchless)** |

This is the same principle that makes [[soa-vs-aos|SoA beat AoS]] and [[csr-graph|CSR beat adjacency lists]]: contiguous layouts and parallel scanning beat pointer-chasing at every level of the cache hierarchy.

## The single-threaded tier list

**Tier 1 — SIMD B-trees (7–18× over `std::map`).** [[algorithmica-s-tree|Algorithmica S+ tree]] (Sergey Slotin) — 7–18× over `std::set` and 3–7× over Abseil on `lower_bound`, in under 150 lines of C++. [[bs-tree|BS-tree]] (ICDE 2026) — 16 keys per node searched in two AVX-512 instructions; gapped nodes for branchless updates. [[fb-plus-tree|FB+-tree]] (VLDB 2025) — extends SIMD B-trees to variable-length keys via byte-wise prefix matching. **frozenca::BTreeSet** — 175 ms vs 498 ms for `std::set` on 1M int64 inserts (2.8× insert, 4× lookup); beats `cpp-btree` by 1.4–1.6×.

**Tier 2 — Production B-tree maps.** [[abseil-flat-hash-map|Google Abseil]] `btree_map` is the gold-standard production default — ~5 bytes/element vs 40 for `std::set<int32_t>`, ~6× fewer cache misses, 1.5–3× over `std::map`. The TLX B+-tree (Timo Bingmann) becomes meaningfully faster than `std::map` once the tree exceeds ~16,000–100,000 items.

**Tier 3 — Database B+-trees.** **LMDB** (Howard Chu) uses memory-mapped B+ trees and dominates read benchmarks against LevelDB, RocksDB, WiredTiger, and BerkeleyDB. The fork **libmdbx** claims 3× faster than BoltDB.

## Concurrent B-trees

For concurrent ordered maps, [[bp-tree|BP-Tree]] (VLDB 2023) is the current state of the art for **range-scan-heavy workloads** — 7.4× over [[masstree]] on point operations and 30× on large range queries. For point operations alone, [[art-olc|ART-OLC]] is faster on dense integer keys; [[masstree]] wins on string keys at high contention. The lock-free [[bw-tree|Bw-tree]] is **1.5–4.5× slower** than all three because lock-freedom does not imply cache-coherence-freedom — the fundamental insight of the 2018 OpenBw-Tree paper.

## Write-optimized variants

[[b-epsilon-tree|Bε-trees]] (fractal tree indexes) trade slightly slower reads for **32× fewer write I/Os** through internal-node update buffering. TokuDB used this in production until its 2022 deprecation; the principle survives in LSM-tree variants. [[cache-oblivious-structures|Cache-oblivious B-trees]] (Bender, Demaine, Farach-Colton) match B-tree bounds without knowing hardware parameters, but tuned cache-conscious designs consistently outperform them in practice.

## Relationship to other tree structures

The [[adaptive-radix-tree|ART]] outperforms B-trees for dense integer keys by eliminating comparisons entirely (O(k) lookup independent of n). [[hot-trie|HOT]] beats ART on string keys with shared prefixes. [[learned-indexes]] beat B-trees on static read-heavy workloads (2–3×) by replacing tree traversals with model evaluations. For the full ordered-map landscape see [[fastest-ordered-maps]].

## The unifying recipe

Pack more keys per cache line. Search them with [[simd-programming|SIMD]]. Eliminate branch mispredictions. Use [[huge-pages|hugepages]] for working sets that exceed a few hundred MB. The same recipe produces the [[swiss-table|fastest hash maps]], the [[lcrq|fastest queues]], and the [[fastest-dynamic-arrays|fastest dynamic arrays]]. On modern hardware, mechanical sympathy for the cache hierarchy *is* the algorithm.
