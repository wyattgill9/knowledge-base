---
tags:
  - rust
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Driftsort

Rust's standard stable sort since 2024, replacing the previous merge sort. Derived from Orson Peters' **glidesort**, driftsort adds robustness guarantees for production use while maintaining glidesort's performance innovations.

## Performance

Glidesort (driftsort's predecessor) achieves up to **4x faster** than Rust's previous `slice::sort` on random data through three key innovations:

- **Lazy logical runs** — instead of eagerly detecting and extending natural runs (like Timsort), glidesort lazily tracks run boundaries and merges only when beneficial.
- **Branchless bidirectional partitioning** — eliminates branch mispredictions during the partition phase, a critical bottleneck on modern CPUs.
- **Interleaved multi-run merging** — merges multiple runs simultaneously, exploiting instruction-level parallelism rather than sequentially merging pairs.

## Context in the sorting landscape

| Need | Best choice |
|------|-------------|
| Sequential unstable | [[pdqsort]] (Rust's `sort_unstable`) |
| Sequential stable | Driftsort (Rust's `sort`) |
| Parallel | [[ips4o]] |

Both [[pdqsort]] and driftsort were created by Orson Peters (driftsort with Lukas Bergdoll). Peters is responsible for two of the three best sorting algorithms in practical use — an extraordinary contribution.

Python 3.11+ took a lighter-touch approach, replacing only Timsort's merge policy with **Powersort** for more balanced merge scheduling while keeping the same underlying adaptive merge-sort framework.
