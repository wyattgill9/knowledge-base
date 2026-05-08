---
tags:
  - data-structures
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Learned Indexes

The most disruptive recent development in ordered-index design. Learned indexes replace traditional tree-based branching with machine-learned models of the data's cumulative distribution function (CDF), achieving **2–3× faster lookups with smaller index sizes** than [[b-tree|B-trees]] for static read-heavy workloads. The idea is elegant: if you know the data distribution, you can predict where a key should be and then do a small local search around the prediction, replacing O(log N) cache-missing tree traversals with one model evaluation plus a tight scan.

## Key implementations

**PGM-Index** (Ferragina & Vinciguerra, 2020) provides O(log log N) lookup in O(N) space using piecewise-linear approximation of the data's CDF. The 2025 **PGM++** variant achieves 2.31× speedup over the original through mixed search strategies.

**RadixSpline** (Kipf et al.) achieved 20% faster reads and 45% less memory than B-trees when integrated into RocksDB — production validation that learned indexes are not just academic.

**LITS** (2024) combines learned models with [[hot-trie|HOT]] tries, claiming **2.4× over HOT** on point operations. This is the current frontier for *string-keyed* learned indexing — the previous learned-index work focused on numerical keys, and the trie integration extends the technique to a much broader workload class.

## Benchmark results

On the SOSD benchmark suite (200M–800M keys), learned models consistently deliver 2–3× faster lookups with smaller index sizes than traditional B-trees for static read-heavy workloads. The advantage comes from replacing O(log N) tree traversals — each potentially a cache miss — with a model evaluation (arithmetic, no cache miss) plus a small local search.

## Limitations

Learned indexes excel at static, read-heavy workloads where the data distribution is stable. They are less suitable for:

- **Write-heavy workloads** — inserting new keys may invalidate the learned model, requiring retraining.
- **Adversarial distributions** — pathological data produces poor predictions and the local search degrades to linear scan.
- **Dynamic datasets** — updatable learned indexes (ALEX, LIPP) exist but add complexity and reduce the performance advantage.

## Where they fit

In the [[fastest-ordered-maps|ordered-map hierarchy]], learned indexes occupy the static-workload tier alongside [[perfect-hashing]] (which beats them on unordered static lookups by eliminating probing entirely). For dynamic workloads with stable distributions, they complement rather than replace [[b-tree|SIMD B-trees]] and [[adaptive-radix-tree|ART]] — RocksDB ships RadixSpline alongside its existing block-cache index, not in place of it.

## Outlook

Learned indexes are already delivering production wins in RocksDB and may fundamentally reshape database indexing. The key insight — that data-dependent structures can outperform data-oblivious ones — extends beyond indexes to filters, compression, and query optimization. Combined with [[cuckoo-trie|memory-level parallelism]] and [[bs-tree|AVX-512 SIMD trees]], the 2024–2026 frontier of ordered indexing is one of the most active areas of database systems research.
