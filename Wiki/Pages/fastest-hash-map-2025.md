---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# The Fastest Hash Map (2025)

The state of the hash map landscape as of 2025. Three concurrent claims define it: [[boost-unordered-flat-map|Boost's `unordered_flat_map`]] is the consensus fastest single-threaded general-purpose hash map, narrowly edging Google's [[abseil-flat-hash-map|`absl::flat_hash_map`]]; both are [[swiss-table]] variants that crush `std::unordered_map` by 2–11×. CMU's [[parlayhash]] reaches 1,130 Mops at 128 threads, an order of magnitude past Meta's folly and Intel TBB. And in January 2025, [[elastic-hashing|Farach-Colton, Krapivin, and Kuszmaul disproved Yao's 1985 conjecture]] on probe complexity, opening new theoretical ground.

## The single-threaded podium

The Jackson Allan benchmark (May 2024) is the most comprehensive recent comparison: 15 implementations, 3 key types, 7 operations, load factor 0.875, AMD Ryzen 7 5800H. Normalized to fastest:

| Operation | boost::unordered_flat_map | absl::flat_hash_map | ankerl::unordered_dense | emilib2 | tsl::robin_map | std::unordered_map |
|---|---|---|---|---|---|---|
| Insert (int→int) | **1.00×** | 1.11× | 1.78× | 1.01× | 1.67× | 6.39× |
| Lookup hit (int) | 1.28× | 1.63× | 1.15× | 1.29× | 1.20× | 2.28× |
| Lookup miss (int) | 1.02× | **1.00×** | 1.33× | 1.26× | 1.55× | 3.67× |
| Erase (int) | 1.36× | 2.17× | 2.77× | 1.33× | 2.47× | 11.26× |
| Iteration (int) | 1.29× | 2.55× | **1.00×** | 1.10× | 1.51× | 3.70× |

Boost wins insertion decisively and dominates erasure thanks to its overflow byte mechanism — an innovation over Abseil's tombstone approach. Where Abseil marks deleted slots with a sentinel that accumulates and degrades probe performance, Boost tracks whether any group has *ever* overflowed. Joaquín M López Muñoz measured 3.2× fewer comparisons on unsuccessful lookups at high load factors. Abseil claws back ground on lookup-miss benchmarks and remains the production reference at Google scale.

## Each winner owns a niche

No hash map wins everywhere. The landscape fractures by workload:

- **[[boost-unordered-flat-map]]** — general-purpose crown. Matches or beats Abseil on insert, erase, and lookup-miss. ~1.07 bytes overhead per bucket. Almost-8-bit reduced hash (7.99 bits vs Abseil's 7).
- **[[abseil-flat-hash-map]]** — large maps with integer keys, the original Swiss Table sweet spot. Still the most widely deployed high-performance hash map in production.
- **[[ankerl-unordered-dense]]** — uncontested iteration champion. Stores all entries in a contiguous `std::vector` (2.55× faster iteration than Abseil). Single-header. Robin Hood probing with backward-shift deletion sidesteps tombstone degradation entirely.
- **[[folly-f14|`folly::F14ValueMap`]]** — lowest memory footprint at 24.7 bytes/element. 14-way SIMD probing with overflow counting (Amble-Knuth 1974).
- **[[emhash|`emhash7`]]** — raw insert/erase throughput; tolerates load factor 0.999 with 3-way linear probing and no tombstones. 3.6 ns/op for integer find-hit (4.2× `std::unordered_map`).
- **[[perfect-hashing|`fph::DynamicFphMap`]]** — ultra-fast lookups on read-only static data; slow construction.

## The Swiss Table revolution spans every major language

The convergence on Swiss Table-derived designs is the defining trend of 2018–2025. See [[swiss-table]] for architecture details.

- **Rust** — [[hashbrown]] became `std::HashMap` in 1.36 (2019). With [[foldhash]] (replacing ahash in 0.15+), it tracks Abseil within ~5%.
- **Go 1.24** (Feb 2025) — rewrote built-in `map` as Swiss Table with zero API changes. 31% faster inserts (137 → 94 ns), 21% faster lookups (41 → 32 ns). Datadog reported 70% memory reduction on large maps; ByteDance measured 20–50% gains on map-heavy queries. Uses 8-slot groups with portable 64-bit integer SIMD emulation rather than hardware SIMD.
- **Java** — bound by boxing (an `Integer` is 16 bytes vs 4 for `int`). Primitive-specialized libraries like Koloboke and fastutil hit 2–4× over `java.util.HashMap`; Eclipse Collections iterates 10× faster. Chronicle Map: 30M updates/sec off-heap with zero GC. A Vector API SwissTable prototype appeared in late 2025 but remains experimental.
- **C#/.NET 8** — [[frozen-dictionary|`FrozenDictionary`]] takes a different angle: an immutable, read-optimized dictionary that analyzes keys at construction. 43–69% faster string lookups. Break-even is 5,000–630,000 reads depending on size.

## The deeper insight: hash function > hash table

The biggest single win in Rust's ecosystem was not a table change. Switching from SipHash (the safe default) to [[foldhash]] yielded **2–5× speedup** in hash-heavy workloads — dwarfing any difference between table implementations. The compiler `rustc` itself got a **6% overall speedup** from switching to fxhash. See [[rapidhash]] for the broader landscape.

This recontextualizes the "fastest hash map" question. For workloads dominated by hashing time (short keys, simple types), the hasher choice swamps the table choice.

## Concurrent maps: ParlayHash leads, but only at scale

For concurrent workloads, [[parlayhash|CMU's ParlayHash]] (2024) is the clear state-of-the-art at high thread counts. On a 128-hyperthread Intel Xeon Ice Lake:

| Implementation | 1 thread (Mops) | 128 threads (Mops) | Memory (B/elt) |
|---|---|---|---|
| **ParlayHash** | 21.4 | **1,130** | 26.3 |
| boost::concurrent_flat_map | **25.3** | 58 | 37.9 |
| folly::ConcurrentHashMap | 12.2 | 157 | 91.9 |
| phmap::parallel_flat_hash_map | 22.0 | 112 | 36.0 |
| libcuckoo | 14.2 | 29 | 43.6 |
| Intel TBB | 13.2 | 54 | — |

ParlayHash's 1,130 Mops is roughly 7× folly, 10× phmap, and 39× libcuckoo. Epoch-based memory reclamation; parallel internal operations via parlaylib. The catch: sequential maps remain far faster single-threaded ([[abseil-flat-hash-map|Abseil]] hits 40.1 Mops alone), so concurrent maps only justify their overhead above ~4–8 threads.

`boost::concurrent_flat_map` (Boost 1.83+) wins single-thread among concurrent maps with the same SIMD-probed layout as its non-concurrent sibling and group-level locks — but collapses past 16 threads as those locks become bottlenecks. In Rust, [[papaya]] leads for read-heavy async; [[dashmap]] and [[scc]] dominate write-heavy. See [[rust-concurrent-data-structures]].

## Why modern hash maps are fast: seven techniques

Modern designs stack reinforcing optimizations:

1. **SIMD group probing** — single most impactful. `_mm_cmpeq_epi8` compares 16 metadata bytes per instruction; ~16× per-probe-step throughput.
2. **Metadata separation** — 16 one-byte H2 fingerprints per cache line act as a Bloom filter, eliminating ~99.2% of false positives before key comparison.
3. **Flat open addressing** — keys and values inline in a contiguous array. Zero per-insert `malloc`. 2–3× speedup on inserts; 100–200× faster clearing.
4. **Power-of-2 sizing** — bitwise AND replaces ~20–40-cycle modulo. Requires a hash that distributes entropy across all bits (wyhash, foldhash, Abseil hash).
5. **High load factor** — Boost and Abseil fix it at 87.5%, far above traditional open addressing, because group probing amortizes collision costs.
6. **Tombstone-free deletion** — Robin Hood backward-shift or Boost's overflow byte. Prevents probe-chain rot under heavy insert/erase cycling.
7. **SIMD-friendly hash function** — see "hash function > hash table" above.

## The theoretical frontier

The most significant academic development is the January 2025 paper by [[elastic-hashing|Farach-Colton, Krapivin, and Kuszmaul]] disproving Yao's 1985 conjecture. Yao had conjectured uniform hashing was optimal for greedy (non-reordering) insertion. Elastic hashing achieves O(log δ⁻¹) worst-case expected probe complexity without reordering — for a 99%-full table, that means ~21 worst-case probes versus the previously believed ~100. A companion *funnel hashing* variant achieves O(1) amortized with matching lower bounds, proving optimality.

[[iceberg-hashing]] (JACM 2023) simultaneously optimizes space, cache efficiency, and stability — important for persistent memory where element movements are expensive. [[folly-f14|Meta's F14]] uses overflow counting (Amble-Knuth 1974) rather than tombstones for production-proven memory efficiency.

## The bottom line

For 2025, the practical answer is clear: **[[boost-unordered-flat-map]]** is the fastest general-purpose hash map, with **[[abseil-flat-hash-map]]** a close second and the better choice within Google's ecosystem. Both implement [[swiss-table]] — the de facto standard across C++, Rust, and Go. The deeper insight is that the hash *function* often matters more than the *table*: foldhash beats SipHash by more than any table-level optimization beats Abseil. For concurrent loads, [[parlayhash]] scales where everything else collapses. For static read-only data, [[perfect-hashing]] (fph, PTHash, RecSplit) remains unbeatable at the cost of construction time. The field is far from settled — [[elastic-hashing]] suggests fundamentally new probing strategies could still yield practical gains.
