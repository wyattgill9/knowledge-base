---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Fastest Ordered Maps in Computer Science

The era of the pointer-chasing binary search tree is over for performance-critical code. SIMD-optimized B-trees outperform red-black trees (`std::map`, Java `TreeMap`) by **7–18× on lookups**, and for dense integer keys the [[adaptive-radix-tree|Adaptive Radix Tree]] matches even tuned B-trees by eliminating key comparisons entirely. The unifying lesson across every tier of this hierarchy: pack more keys per cache line, search them with [[simd-programming|SIMD]], and eliminate branch mispredictions. On modern hardware, the memory hierarchy is the algorithm.

## Why pointer-chasing trees lost

A single DRAM cache miss costs **60–100 ns** (~150–300 cycles); an L1 hit costs **~1 ns**. A red-black tree at 1M keys is ~20 levels deep and chases a pointer at every level — almost every comparison is a cache miss. A B-tree with fanout 62 collapses that to **3–4 levels**, and SIMD scans every node in a single clock cycle. This compounds:

| Factor | `std::map` (red-black) | [[abseil-flat-hash-map\|Abseil]] `btree_map` | SIMD B-tree (Algorithmica) |
|--------|------------------------|----------------------------------------------|-----------------------------|
| Tree height (1M int32) | ~20 | ~3–4 | ~3 |
| Bytes per element | 40 | ~5 | ~5.5 |
| Cache misses / lookup | ~20 | ~4 | ~3 |
| Intra-node search | scalar, 1 key | linear, ~30 keys | **SIMD, 8+ keys/cycle** |
| Branch mispredicts / lookup | ~10 | ~3–4 | **~0 (branchless)** |

No amount of algorithmic cleverness within the binary-tree shape escapes this. The Ω(log n) comparison-based lower bound is theoretically still binding, but constants now decide every benchmark.

## The single-threaded hierarchy

**Tier 1 — SIMD-optimized B-trees (7–18× over `std::map`).** [[algorithmica-s-tree|Algorithmica's S+ tree]] (Sergey Slotin) achieves 7–18× on `lower_bound` over `std::set` and 3–7× over Abseil's `btree_set`, in under 150 lines of C++. The newer [[bs-tree]] (ICDE 2026) packs **16 keys per node, searched in two AVX-512 instructions**, with gapped nodes for branchless updates. [[fb-plus-tree|FB+-tree]] (VLDB 2025) extends SIMD B-trees to variable-length keys via byte-wise prefix matching with AVX-512.

**Tier 2 — Specialized B-tree libraries (2–4×).** `frozenca::BTreeSet` benchmarks at 175 ms for 1M random int64 inserts vs. ~498 ms for `std::set` (2.8× insert, 4× lookup), and beats Google's `cpp-btree` by 1.4–1.6×. `kressler/fast-containers` claims 2–5× over Abseil with [[huge-pages|hugepage]] allocation contributing 3–5× on its own once the working set passes ~1 GB.

**Tier 3 — Production B-tree maps (1.5–3×).** [[abseil-flat-hash-map|Google's Abseil]] `btree_map` is the gold standard: ~5 bytes per element vs. 40 for `std::set<int32_t>`, 6× fewer cache misses, and the correct production default. The TLX B+-tree (Timo Bingmann) becomes meaningfully faster than `std::map` once the table exceeds ~16,000–100,000 items.

**Tier 4 — Standard library defaults.** `std::map`, Java `TreeMap`, .NET `SortedDictionary`. Per-node heap allocation, 40-byte overhead per element, one key per cache line — the slowest mainstream ordered map by a wide margin.

## ART and tries: where comparisons disappear

For dense integer keys, [[adaptive-radix-tree|ART]] achieves **O(k) lookup independent of n** — for 8-byte integers that's a fixed 8 byte lookups whether the tree holds 1,000 or 1 billion entries. ART matches hash-table speeds at only 8.1 bytes per key on dense datasets while preserving sorted order. [[hot-trie|Height Optimized Tries]] (HOT, Binna et al., SIGMOD 2018) extend the advantage to string keys by varying bits per node based on data distribution, beating ART, B-trees, and Masstree on string workloads. The [[cuckoo-trie]] (2021) issues independent memory accesses per lookup to exploit **memory-level parallelism**, outperforming state-of-the-art indexes by 20–360% with a smaller footprint.

Where B-trees still win: long string keys with shared prefixes (each trie level may pay a cache miss for one byte of discrimination) and range scans (trie iteration is less cache-friendly than B+-tree leaf scanning). For arbitrary-key general-purpose maps, B-trees remain the better default.

## Concurrent ordered maps: optimistic locking beats lock-freedom

The OpenBw-Tree paper (Wang et al., SIGMOD 2018) overturned conventional wisdom: the lock-free [[bw-tree|Bw-tree]] was **1.5–4.5× slower** than lock-based alternatives. The 2025 ranking:

1. **[[art-olc|ART with Optimistic Lock Coupling]]** — fastest for point operations. Readers never write shared memory, eliminating cache-line invalidation traffic. [[congee|Congee]] (Rust ART-OLC) hits **150 Mops/sec on 32 cores** with 8-byte keys; 4.8× fewer instructions and 3.6× fewer L3 misses than lock-coupled B+-trees.
2. **[[masstree]]** — best under high contention and for string keys. Trie-of-B+-trees architecture; 6–10 Mops/sec on 16 cores under YCSB; only 2.5× slower than a near-optimal hash table while preserving range queries.
3. **B+-tree with OLC** — dominant for range scans. [[bp-tree|BP-Tree]] (VLDB 2023) hits 7.4× over Masstree on point operations and 30× on large range queries. B-skiplists (ICPP 2025) reach 2–9× over traditional concurrent skip lists by packing nodes.
4. **Java's `ConcurrentSkipListMap`** — simplest correct concurrent ordered map; loses to `TreeMap` single-threaded but scales under contention. Bronson's SnapTree (concurrent AVL) showed 39% higher throughput than `ConcurrentSkipListMap`.

The principle behind the upset: **minimizing cache-line invalidation matters more than eliminating locks**. Bw-tree's delta chains add pointer chasing; its CAS-heavy design generates coherence traffic. ART-OLC and Masstree readers do *zero writes* to shared memory, achieving near-lock-free scalability with dramatically simpler code. This is the same insight that makes [[faa-vs-cas|FAA beat CAS]] in queues and that makes [[rcu]] unbeatable in read-dominated workloads.

## Theoretical sub-logarithmic structures stay theoretical

Three structures beat O(log n) on integer keys, none of them practical:

- **[[van-emde-boas-tree|Van Emde Boas trees]]** — O(log log U) per op (~6 ops for 64-bit keys), but O(U) space without hashing and large constants. Practical implementations exist; they rarely beat tuned B-trees on real hardware.
- **[[fusion-tree|Fusion trees]]** — O(log n / log log n) via word-level parallelism (~4× height reduction over binary search). The bit manipulation is so complex that no practical implementation competes with straightforward B-trees.
- **Exponential search trees** (Andersson & Thorup) — O(√(log n / log log n)) deterministic worst case. Pătrașcu and Thorup proved this is near-optimal for linear-space predecessor structures. Purely theoretical.

In the external-memory model, B-trees are I/O-optimal at O(log_B N). [[b-epsilon-tree|Bε-trees]] (fractal tree indexes) trade slightly slower reads for **32× fewer write I/Os** at typical parameters; TokuDB used this in production until its 2022 deprecation, and the principle lives on in LSM variants.

[[cache-oblivious-structures|Cache-oblivious B-trees]] match B-tree bounds without knowing hardware parameters, but tuned cache-conscious designs consistently outperform them. The lesson is consistent: theoretical optimality and practical winning are different competitions.

## The 2024–2026 frontier

Three developments are reshaping the landscape:

**AVX-512 is now ubiquitous.** AMD Zen 5 (2024) provides native 512-bit execution (not double-pumped); Intel Granite Rapids delivers full AVX-512 throughput. B-tree nodes with 16 × 32-bit keys searched in two instructions — the [[bs-tree]] design — are now portable rather than Intel-specific.

**Learned indexes meet tries.** [[learned-indexes|Learned indexes]] replace tree levels with lightweight ML models predicting key positions. **LITS** (2024) combines learned models with tries, claiming 2.4× over [[hot-trie|HOT]] for point operations.

**3D V-Cache extends in-memory limits.** AMD's stacked-cache CPUs reach 96–128 MB of L3, raising the threshold at which tree indexes spill to DRAM. A B-tree with fanout 64 holds **billions of keys** entirely in L3 cache on these processors — the regime where the SIMD-B-tree advantage is largest.

## Decision guide

| Workload | Pick |
|----------|------|
| Read-heavy integer keys, no portability constraint | [[bs-tree]] / [[algorithmica-s-tree]] (50–100M+ qps single-core) |
| Production C++ default | [[abseil-flat-hash-map\|Abseil]] `btree_map` |
| Dense integer keys / in-memory database | [[adaptive-radix-tree\|ART]] |
| String keys, single-threaded | [[hot-trie\|HOT]] |
| Variable-length keys, SIMD path | [[fb-plus-tree\|FB+-tree]] |
| Concurrent point ops | [[art-olc\|ART-OLC]] / [[congee]] |
| Concurrent range scans | [[bp-tree\|BP-Tree]] |
| Concurrent string keys, high contention | [[masstree]] |
| Static read-only | [[learned-indexes]] / [[perfect-hashing]] |
| Write-amplification-bound external memory | [[b-epsilon-tree\|Bε-tree]] / LSM |

## The unifying principle

Every winner — SIMD B-trees, ART, HOT, ART-OLC, Masstree — follows the same recipe: **pack more keys per cache line, search them with SIMD, eliminate branch mispredictions, and avoid writes to shared cache lines**. This is the same recipe that produces the [[fastest-hash-map-2025|fastest hash maps]] ([[swiss-table]] metadata scanning), the [[the-fastest-queue|fastest queues]] ([[faa-vs-cas|FAA-on-a-ring]]), and the [[fastest-dynamic-arrays|fastest dynamic arrays]] (allocator-aware in-place expansion). On modern hardware, mechanical sympathy for the cache hierarchy is the algorithm.
