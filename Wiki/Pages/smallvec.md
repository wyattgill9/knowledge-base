---
tags:
  - rust
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# SmallVec

`SmallVec<[T; N]>` stores up to N elements inline on the stack before falling back to heap allocation. It is the standard Rust answer for collections that are usually small (3–8 elements) but occasionally grow — avoiding heap allocation entirely on the common path. Part of the [[rust-allocation-patterns|borrow → Cow → owned]] hierarchy and the canonical example of [[small-buffer-optimization|SBO]] in the Rust ecosystem.

## The empirical case is weaker than it looks

The intuition that "skip malloc when small" must win is not borne out by benchmarks. For `from_slice` operations, `Vec` with pre-allocated capacity (`Vec::with_capacity(n)`) was measured **up to 5.5× faster than `SmallVec`** on the same workload. The benchmark author's conclusion: "malloc is fast enough that SmallVec's increased complexity often hurts more than its allocation savings help."

The mechanism is the same as with C++ SBO containers ([[absl-inlinedvector]], [[llvm-smallvector]]): every access pays a branch checking inline vs. heap mode, and that branch interferes with compiler optimizations the simpler `Vec` benefits from. Random access, iteration, and bulk loads are all penalized — sometimes by enough to dwarf the malloc savings.

## When SmallVec actually wins

Three scenarios:

1. **Hot loops creating and destroying many small vectors.** When the malloc/free pair is the dominant cost, eliminating it pays for the per-access branch many times over.
2. **`Vec<SmallVec<[T; N]>>` containers-of-containers.** Inline buffers stay contiguous in the outer allocation, giving one block instead of fragmented small heap allocations.
3. **Fragmentation-sensitive long-running services.** Less malloc churn means less fragmentation pressure on the global allocator.

For the much more common "I might have a few elements" case, `Vec` with `with_capacity` or even the default `Vec::new()` plus a [[mimalloc|good allocator]] is the right answer. The malloc cost is small; the branch cost is not.

## When the upper bound is fixed

If the maximum size is known at compile time, `arrayvec::ArrayVec` or [[heapless|`heapless::Vec`]] eliminate even the inline/heap branch — true O(1) non-amortized push, no fallback. This is structurally similar to C++'s [[static-vector|`boost::container::static_vector`]] and is strictly better than SmallVec when the bound is genuinely known.

See [[small-buffer-optimization]] for the cross-language SBO landscape and [[fastest-dynamic-arrays]] for the broader hierarchy of dynamic-array optimizations.
