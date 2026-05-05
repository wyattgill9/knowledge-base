---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# absl::InlinedVector

Google's `absl::InlinedVector<T, N>` is a [[small-buffer-optimization|small-buffer-optimized]] vector designed as a perfect drop-in for `std::vector` — same API, same iterator semantics, with an inline buffer of size N that defers heap allocation. Unlike [[llvm-smallvector|LLVM's SmallVector]], it does not provide a type-erased base; every distinct N value produces a distinct type.

## The design choice

Abseil prioritizes API parity. `absl::InlinedVector<int, 4>` should behave identically to `std::vector<int>` from the user's perspective, modulo allocation behavior. Existing code using `std::vector` should compile and run correctly when the type is swapped. The cost of this commitment is no `SmallVectorImpl`-style erasure: a function taking `absl::InlinedVector<T, 4>&` cannot accept `absl::InlinedVector<T, 8>&`. For Google's typical use case — picking one N per declaration site rather than passing vectors across function boundaries — this is acceptable.

## The random-access surprise

The single most striking benchmark from the dynamic-array survey: random access on `absl::InlinedVector` was measured **~5× slower** than `std::vector`. The expected slowdown from the inline/heap branch should be at most ~2×; 5× is anomalous. The most plausible explanation is that the branch interferes with compiler optimizations the simpler `std::vector` benefits from — alias analysis, vectorization, hoisting — so the realized cost is much higher than the static instruction cost suggests.

This is a recurring lesson in SBO containers: benchmarks dominate intuition. The branch is "cheap" in isolation but expensive in context, and the context is what programs actually do.

## When to reach for it

Pick `absl::InlinedVector` when (a) your team is already in the Abseil ecosystem, (b) you have a clear hot-loop case where a small-N path matters, and (c) you have measured that the inline path actually wins. Avoid it as a default replacement for `std::vector` — the random-access regression alone is enough to make it a worse choice for general-purpose code.

For library design where vectors cross function boundaries, [[llvm-smallvector|SmallVector]] is more practical. For minimum object size, [[ankerl-svector]] beats it. For fixed bounds, [[static-vector|static_vector]] is strictly better. See [[small-buffer-optimization]] for when SBO is the right tool and [[fastest-dynamic-arrays]] for the broader ranking.
