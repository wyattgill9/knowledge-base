---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# folly::small_vector

Facebook's `folly::small_vector` takes [[small-buffer-optimization|SBO]] to its memory-efficiency limit. The inline/heap discriminator is packed into the **highest bit of the size field**, giving a single-bit overhead for the entire SBO mechanism. The container is designed for workloads with billions of small vectors, where every byte per instance translates into hundreds of megabytes of total memory.

## The bit-packing trick

A traditional SBO container needs a flag (or pointer-tag, or invariant violation) to distinguish inline from heap mode. The cheapest representation typically uses a byte or steals the low bit of a pointer. `folly::small_vector` instead packs the flag into the *high* bit of the size counter — assuming sizes never exceed 2^63 (or 2^31 on 32-bit), the high bit is always zero in valid sizes and can carry one bit of mode information.

The result: `sizeof(folly::small_vector<T, N>)` is smaller than competing SBO containers by 4–8 bytes per instance. For the workloads Folly was designed around — long-running services holding billions of `small_vector<Edge, 2>` graph adjacency lists or `small_vector<Field, 4>` schema records — that adds up to gigabytes of saved RAM at fleet scale.

## When the design pays off

The bit-packing optimization only matters when:

- You have *billions* of small_vector instances live simultaneously.
- Per-instance memory is the dominant cost (not allocation throughput, not iteration speed).
- The size domain genuinely never approaches 2^63 — true for any realistic case.

For typical "I have a few hundred small_vectors" workloads, the byte savings are irrelevant and the additional bit-extraction logic on every size query adds overhead. Pick [[absl-inlinedvector|`absl::InlinedVector`]] for `std::vector`-like ergonomics, [[llvm-smallvector|`llvm::SmallVector`]] for type-erased function signatures, or plain `std::vector` with a [[mimalloc|good allocator]] for general-purpose code.

## The Folly philosophy

`folly::small_vector` follows the same pattern as the rest of Folly: extreme optimization for Facebook-scale workloads, with the assumption that the hosting application has done the profiling to know it actually needs this. It pairs naturally with [[folly-fbvector]] (for vectors that grow over time) and [[folly-f14]] (for hash maps with similar memory-density priorities).

See [[small-buffer-optimization]] for SBO in general and [[fastest-dynamic-arrays]] for when SBO is and is not the right tool.
