---
tags:
  - performance
  - concurrency
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# IPS⁴o (In-Place Super Scalar Samplesort)

The clear winner for parallel sorting across the broadest set of benchmarks. Developed at KIT (Karlsruhe Institute of Technology), IPS⁴o was tested across 21 sorting implementations, 6 data types, 10 distributions, 4 machines, and 7 orders of input magnitude — and outperformed everything.

## Performance

IPS⁴o beats the closest sequential competitor (BlockQuicksort) by **1.5x** and the closest in-place parallel competitor by **3x**. It even beats dedicated integer sorting algorithms in many configurations, which is remarkable for a comparison-based sort.

## Design

IPS⁴o's key innovations are block-based partitioning that avoids branch mispredictions and achieves excellent cache behavior. The "super scalar" part refers to exploiting instruction-level parallelism — keeping multiple pipeline stages busy simultaneously. The "in-place" part means O(log n) auxiliary space, unlike merge-sort-based parallel algorithms that require O(n) extra memory.

## The practical sorting stack

| Need | Algorithm |
|------|-----------|
| Sequential unstable | [[pdqsort]] |
| Sequential stable | [[driftsort]] |
| Parallel (any) | IPS⁴o |
| Integer-specific | IPS²Ra or ska_sort |

IPS⁴o is the right choice whenever you have multiple cores and enough data to amortize the parallelism overhead.
