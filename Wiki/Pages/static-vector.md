---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# static_vector

`boost::container::static_vector<T, N>` is a fixed-capacity vector with **zero heap allocation** and zero inline/heap branching. It stores up to N elements in inline storage and refuses (asserts or throws) on overflow. Push_back is genuinely O(1) — not amortized O(1) — because there is no growth path. The proposal P0843 is moving this into the C++ standard as `std::static_vector`.

## Why it beats SBO containers

[[small-buffer-optimization|SBO]] containers like [[llvm-smallvector|SmallVector]] and [[absl-inlinedvector|InlinedVector]] pay a branch on every access to discriminate between inline storage and heap fallback. `static_vector` eliminates the branch entirely because there is no fallback. The compiler sees a fixed-capacity buffer and generates straight-line code: increment size, construct element, done. For workloads where the upper bound is genuinely known at compile time, this is strictly better than any SBO design.

The cost is rigidity. Pushing past N is a hard error, not a slow path. This is appropriate for cases like "at most 8 children in this AST node" or "at most 16 candidates in this ranker" where the bound is structural, not statistical.

## When to reach for it

- Bounded buffers with a known compile-time maximum.
- Hot paths where every branch matters and the fallback would be a bug anyway.
- Embedded and kernel code where heap allocation is forbidden — see [[heapless]] for the Rust analog.
- Containers-of-containers where you want guaranteed contiguous layout.

When the bound is statistical rather than structural — most calls are small but occasionally a request has 10,000 elements — SBO is the right answer. When the bound is structural, `static_vector` is.

## In Rust

The closest Rust analog is `arrayvec::ArrayVec<T, N>` or `heapless::Vec<T, N>` (no_std). Both eliminate the heap fallback and the branch overhead. See [[heapless]] for the const-generics-based stack-allocated collection family.

See [[fastest-dynamic-arrays]] for how `static_vector` fits in the broader vector landscape and [[small-buffer-optimization]] for why eliminating the heap fallback often beats embracing it.
