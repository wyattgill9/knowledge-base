---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Small Buffer Optimization

Small buffer optimization (SBO) — also called small-size optimization (SSO) for strings — embeds a fixed inline buffer inside a container so that small instances avoid heap allocation entirely. The technique looks obvious on paper: heap allocation is expensive, so put the data inline until you must spill. In practice the empirical case is far weaker than the intuition suggests, and SBO is one of the most overrated optimizations in the C++ landscape.

## The mechanics

A typical SBO vector stores three things: a pointer (or inline buffer), a size, and a capacity. The discriminator between inline and heap mode is either a flag bit, the relationship between size and the inline-buffer threshold, or a self-referential pointer trick. On every access, the container must check which mode it is in — that branch is the cost of admission.

Inline storage gets you (a) zero malloc cost when small, (b) all element data in a single cache line, (c) trivial relocation when the moved-from value fits inline. Heap fallback gets you the standard `std::vector` semantics.

## The C++ landscape

| Container | Inline default | Distinguishing feature |
|-----------|---------------|------------------------|
| [[llvm-smallvector\|`llvm::SmallVector`]] | Auto-tuned to ~64 B sizeof | `SmallVectorImpl<T>` type erasure for template bloat avoidance |
| [[absl-inlinedvector\|`absl::InlinedVector`]] | User-specified N | Perfect `std::vector` API drop-in |
| [[folly-small-vector\|`folly::small_vector`]] | User-specified N | Inline/heap flag in highest bit of size; designed for billions of vectors |
| [[ankerl-svector\|`ankerl::svector`]] | 7 inline bytes | 8-byte minimum sizeof; smallest possible footprint |
| [[smallvec\|Rust `SmallVec`]] | User-specified N | Standard Rust ecosystem entry |

## The unflattering benchmarks

For 1000-int push_back, plain `std::vector` actually wins because SBO containers pay a branch on every access (checking inline vs. heap). For random access on `absl::InlinedVector`, the overhead was measured **~5× slower** than `std::vector` — an extreme result likely caused by the inline/heap check interfering with compiler optimizations the simpler structure benefits from.

Rust's SmallVec benchmarks showed `Vec` with pre-allocated capacity was up to **5.5× faster** than `SmallVec` for `from_slice`. The benchmark author's conclusion was direct: "malloc is fast enough that SmallVec's increased complexity often hurts more than its allocation savings help." See [[smallvec]] for the Rust-specific picture.

## When SBO actually wins

Three scenarios:

1. **Hot loops creating and destroying many small vectors.** The dominant cost becomes the malloc/free pair, which SBO eliminates entirely.
2. **Containers of containers.** `vector<small_vector<int, 4>>` keeps every inline buffer contiguous in the outer allocation, giving you one contiguous block instead of a fragmented forest of tiny heap allocations.
3. **Memory-fragmented environments.** Long-running servers where heap fragmentation matters more than peak speed.

For most "I might have a few elements" cases, the right answer is `std::vector` with a [[mimalloc|good allocator]], or — if the upper bound is genuinely fixed — [[static-vector|`boost::container::static_vector`]], which eliminates the inline/heap branch entirely.

## The cleaner alternative for fixed bounds

If your size is bounded by a known constant, [[static-vector|`static_vector`]] is strictly better than SBO: no heap fallback, no branch, true O(1) non-amortized push_back. P0843 standardizes it as `std::static_vector`. SBO is the right answer only when you genuinely need the heap fallback for the rare large case but want the common-small case to be free.

See [[fastest-dynamic-arrays]] for SBO's place in the broader vector hierarchy.
