---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Fusion Tree

Fredman & Willard, 1990. Achieves **O(log n / log log n)** per operation by using **word-level parallelism** to compare multiple keys per machine word — packing several "sketches" of keys into a single 64-bit register and operating on them simultaneously with carefully constructed bit tricks. With 64-bit words, this delivers roughly a 4× tree-height reduction over comparison-based binary search.

## Why it stays theoretical

The bit manipulation required to make fusion trees work — the sketch construction, the parallel comparison via multiplication, the most-significant-set-bit operations — is dense and difficult to implement correctly. **No practical implementation has been shown to compete with straightforward [[b-tree|B-trees]]**: by the time the fusion-tree machinery resolves what a [[bs-tree|SIMD B-tree]] resolves in two AVX-512 instructions, the B-tree has already finished. The asymptotic win is real; the constants are catastrophic.

Fusion trees also produced an important *theoretical* contribution by proving sub-logarithmic predecessor search is achievable in the word-RAM model — leading to the [[van-emde-boas-tree|Van Emde Boas]] / fusion / exponential-search-tree family of bounds.

## The lesson

Fusion trees are the best-known case of the gap between the word-RAM model and real hardware. SIMD instructions provide *exactly* the word-parallelism fusion trees postulated, except for arbitrary integer comparison rather than the exotic bit-trick comparison fusion trees needed. Modern [[bs-tree|SIMD B-trees]] capture the available parallelism with a clean implementation and outperform what fusion trees could have achieved even in principle, given the constant-factor reality. See [[fastest-ordered-maps]].
