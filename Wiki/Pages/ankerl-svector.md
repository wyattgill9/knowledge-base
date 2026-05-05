---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# ankerl::svector

Martin Leitner-Ankerl's `ankerl::svector` is a single-header [[small-buffer-optimization|SBO]] vector achieving a remarkable 8-byte minimum `sizeof` — 7 bytes of inline storage and 1 byte of overhead. By comparison, [[absl-inlinedvector|`absl::InlinedVector`]], [[llvm-smallvector|`llvm::SmallVector`]], and [[folly-small-vector|`folly::small_vector`]] all sit at 24–32 bytes minimum. For workloads dominated by per-instance memory cost, this difference is the largest in the SBO landscape.

## The 8-byte design

A pointer-and-size-and-capacity vector is 24 bytes on 64-bit (3 × 8 B). SBO containers add an inline buffer, pushing total size to 32 bytes or more. `ankerl::svector` collapses the representation into 8 bytes by:

1. Storing inline data and the heap pointer in the same 8-byte slot — a `union`.
2. Encoding the discriminator and size in a single byte at the end, leaving 7 bytes for inline payload.
3. Pushing all the bookkeeping (heap size, heap capacity) into the heap allocation itself when the vector spills.

The result is a vector whose footprint matches a single pointer until it overflows. For containers-of-containers (like a `vector<svector<char>>` for storing tags or short strings), this is transformative — a million such vectors cost 8 MB total, vs. 24–32 MB for the standard SBO designs.

## When 8 bytes matters

The same scenarios that motivate [[folly-small-vector]] but with even tighter memory budgets:

- Memory-pressure systems where every byte of per-instance overhead multiplies across hundreds of millions of instances.
- Hot data structures where `vector<svector<T>>` needs to fit working sets in L2 or L3 cache.
- Embedded or edge contexts with strict memory limits.

The trade is that 7 inline bytes is enough for very few element types. `svector<int>` holds 1 element inline; `svector<char>` holds 7. Past that, every push spills to heap. For the cases where N=1 or N=7 is the right inline count, this is exactly the right design.

## Positioning

`ankerl::svector` and [[ankerl-unordered-dense]] both come from the same author with the same philosophy: single-header, microbenchmark-driven, willing to make aggressive design choices that mainstream containers cannot. They are not drop-in standard-library replacements; they are tools for hot paths where the standard answer is too generic.

See [[small-buffer-optimization]] for the full SBO landscape and [[fastest-dynamic-arrays]] for where SBO sits in the dynamic-array hierarchy.
