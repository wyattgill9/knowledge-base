---
tags:
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# pdqsort (Pattern-Defeating Quicksort)

The fastest sequential unstable sorting algorithm in practice. Created by Orson Peters, pdqsort combines randomized quicksort's average case with heapsort's worst-case guarantee while achieving linear time on sorted, reverse-sorted, and equal-element inputs.

## Design

pdqsort is an introsort variant that detects patterns in the input and adapts its strategy. On random data it behaves like quicksort. On sorted or nearly-sorted data it achieves linear time. On adversarial inputs it falls back to heapsort for O(n log n) worst case. The `pdqsort_branchless` variant uses BlockQuicksort's technique to eliminate branch mispredictions on large arrays — a critical optimization on modern CPUs where branch mispredictions cost 15–20 cycles each.

## Adoption

Rust adopted pdqsort as `slice::sort_unstable`, making it the default unstable sort for the entire Rust ecosystem. It is also available in Boost C++ Libraries. Peters later created [[driftsort]] (derived from [[driftsort|glidesort]]) which became Rust's stable sort.

## Relationship to other sorts

For parallel sorting, [[ips4o]] dominates — 1.5x faster than pdqsort sequentially and 3x faster than the closest in-place parallel competitor. For stable sorting, [[driftsort]] is the state of the art. For integer data specifically, IPS²Ra and ska_sort provide radix-sort alternatives. Python 3.11+ uses Powersort (improved Timsort merge scheduling) for its stable sort.

The three algorithms form a complete practical sorting toolkit: pdqsort for sequential unstable, [[driftsort]] for sequential stable, [[ips4o]] for parallel.
