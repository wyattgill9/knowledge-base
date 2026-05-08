---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Van Emde Boas Tree

The classical sub-logarithmic ordered map: **O(log log U)** per operation where U is the universe size. For 64-bit integer keys that's roughly six operations per query — strictly better than the Ω(log n) comparison-based lower bound by exploiting the fact that integer keys *are* their own bit patterns and need not be compared via a black-box comparator.

## Why it's not the answer

Two reasons it loses to tuned [[b-tree|B-trees]] in practice:

1. **Space.** The straightforward construction uses **O(U)** space — for 64-bit keys that's 2⁶⁴ entries, obviously infeasible. Hashed variants drop the cost, but at meaningful constant overhead.
2. **Constants.** The recursive Van Emde Boas structure performs lots of indirect lookups per "operation." On modern hardware, six pointer-chasing operations easily costs more than three SIMD-scanned [[b-tree|B-tree]] node visits, despite the asymptotic win.

Implementations exist and are sometimes used for very-large-universe predecessor problems, but they rarely outperform tuned B-trees on realistic hardware. The [[fastest-ordered-maps|fastest ordered map]] is whatever has the best constants for the cache hierarchy, not the structure with the best big-O.

## Theoretical neighbors

The companion sub-logarithmic structures from the predecessor-search literature:

- [[fusion-tree|Fusion trees]] — O(log n / log log n) via word-level parallelism
- Exponential search trees (Andersson & Thorup) — O(√(log n / log log n)) deterministic worst case; Pătrașcu and Thorup proved this near-optimal for linear-space predecessor structures

All three remain primarily theoretical. The [[cache-oblivious-structures|cache-oblivious B-tree]] proves that B-tree bounds are achievable without knowing hardware parameters — the practical loss has nothing to do with O(log n) being too slow and everything to do with cache misses being expensive. See [[fastest-ordered-maps]].
