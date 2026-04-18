---
tags:
  - data-structures
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Learned Indexes

The most disruptive recent development in data structure design. Learned indexes replace traditional tree-based branching with machine-learned models of the data's cumulative distribution function (CDF), achieving 2–3x faster lookups with smaller index sizes than [[b-tree]] for static read-heavy workloads.

## Key implementations

**PGM-Index** (Ferragina & Vinciguerra, 2020) provides O(log log N) lookup in O(N) space using piecewise linear approximation of the data's CDF. The 2025 **PGM++** variant achieves 2.31x speedup over the original through mixed search strategies. The idea is elegant: if you know the data distribution, you can predict where a key should be and then do a small local search around the prediction.

**RadixSpline** (Kipf et al.) achieved 20% faster reads and 45% less memory than B-trees when integrated into RocksDB — a production validation that learned indexes are not just academic.

## Benchmark results

On the SOSD benchmark suite (200M–800M keys), learned models consistently deliver 2–3x faster lookups with smaller index sizes than traditional B-trees for static read-heavy workloads. The advantage comes from replacing O(log N) tree traversals (each requiring a cache miss) with a model evaluation (arithmetic, no cache miss) plus a small local search.

## Limitations

Learned indexes excel at static, read-heavy workloads where the data distribution is stable. They are less suitable for:

- **Write-heavy workloads** — inserting new keys may invalidate the learned model, requiring retraining.
- **Adversarial distributions** — if the data distribution is pathological, the model provides poor predictions and the local search degrades to linear scan.
- **Dynamic datasets** — though updatable learned indexes exist (ALEX, LIPP), they add complexity and reduce the performance advantage.

## Outlook

Learned indexes are already delivering production wins in RocksDB and may fundamentally reshape database indexing. The key insight — that data-dependent structures can outperform data-oblivious ones — extends beyond indexes to filters, compression, and query optimization. This is one of the most active areas of database systems research.
