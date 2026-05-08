---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Adaptive Radix Tree (ART)

The standout novel tree structure for in-memory ordered indexing on integer or fixed-width byte-string keys. Introduced by Viktor Leis at TU Munich (ICDE 2013), ART decomposes keys byte-by-byte through a trie with **four adaptive node sizes**, each optimized for a different fanout. For dense integer keys it matches hash table speeds while preserving sorted order — and at only **8.1 bytes per key** of overhead on dense data.

## Lookups in O(k), not O(log n)

ART's defining property is that lookup time depends only on **key length**, not tree size. For 8-byte integer keys, every lookup is exactly 8 byte-step descents whether the tree holds 1,000 keys or 1 billion. This breaks the comparison-based Ω(log n) lower bound by exploiting the fact that fixed-width keys have a fixed bit-budget — the same observation that powers [[van-emde-boas-tree|Van Emde Boas trees]] but with vastly better constants.

## Node types

| Node type | Children | Lookup strategy |
|-----------|----------|----------------|
| Node4 | 1–4 | Linear scan |
| Node16 | 5–16 | **SSE comparison of all 16 keys in one instruction** |
| Node48 | 17–48 | 256-byte index array → O(1) child lookup |
| Node256 | 49–256 | Direct array indexing |

Node16 is the [[simd-programming|SIMD]] highlight: a single SSE compare resolves the next child, the same technique that powers [[swiss-table]] metadata scanning. Node48 trades a 256-byte index for O(1) lookup with no comparisons. The adaptive layout means nodes pay for fanout they actually use rather than provisioning for the worst case.

## Optimizations

**Path compression** collapses single-child chains into stored prefixes, dramatically reducing tree height for keys with shared prefixes. **Lazy expansion** defers internal node creation until a split is actually needed, keeping memory low under sequential inserts.

## Concurrent ART

[[art-olc|ART with Optimistic Lock Coupling]] is the **fastest concurrent ordered map for point operations** on dense integer keys. Readers verify version counters rather than acquiring locks, never writing to shared cache lines and therefore never triggering coherence traffic. ART-OLC achieves 4.8× fewer instructions and 3.6× fewer L3 cache misses than lock-coupled B+-trees; the Rust port [[congee]] hits 150 Mops/sec on 32 cores. The lock-free [[bw-tree]] *loses* to ART-OLC by 1.5–4.5× because lock-freedom does not imply cache-coherence-freedom.

## Where B-trees still win

ART struggles with **long string keys sharing common prefixes** (each trie level may pay a cache miss for one byte of discrimination) and **range scans** (trie iteration is less cache-friendly than B+-tree leaf scanning). [[hot-trie|HOT]] addresses the long-string case by varying bits per node based on data distribution. For arbitrary key types in a general-purpose ordered map, [[b-tree|B-trees]] remain the better default. For variable-length keys with SIMD, [[fb-plus-tree]] is the modern alternative.

## Memory-level parallelism descendant

The [[cuckoo-trie]] (2021) extends the trie idea by issuing independent memory accesses per lookup to exploit out-of-order CPUs' memory parallelism, outperforming state-of-the-art indexes by 20–360% with smaller footprint. The principle generalizes — any structure that produces independent miss streams beats one that serializes them.

## Production use

ART powers the **HyPer database system**. [[learned-indexes|Learned indexes]] now compete on static workloads — and **LITS** (2024) combines learned models with [[hot-trie|HOT]] tries for 2.4× over HOT. For the full ordered-map landscape see [[fastest-ordered-maps]].

## Relationship to other structures

ART competes with [[b-tree|B-trees]] for ordered indexing but excels on dense integer or fixed-prefix byte-string keys. For string-keyed indexing without SIMD-tree machinery, [[hot-trie|HOT]] is faster. For immutable string sets at minimum space, [[fst-finite-state-transducer|FSTs]] win. For unordered access only, [[swiss-table]] variants win on point throughput.
