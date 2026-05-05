---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# llvm::SmallVector

LLVM's `SmallVector<T, N>` is a [[small-buffer-optimization|small-buffer-optimized]] vector with one design choice that distinguishes it from every other SBO container: a type-erased base class `SmallVectorImpl<T>` independent of the inline capacity `N`. Function signatures take `SmallVectorImpl<T>&`, allowing call sites to pass `SmallVector<T, 4>` and `SmallVector<T, 16>` interchangeably. This prevents combinatorial template instantiation across LLVM's massive codebase.

## The template-bloat problem

A naive SBO vector parameterized by `<T, N>` produces a separate type for every `N` value. A function `void process(SmallVector<int, 8>& v)` cannot accept a `SmallVector<int, 16>`. Worse, every call site duplicates the implementation in the binary. For a project with thousands of files and hundreds of typedefs, the resulting code bloat is prohibitive.

LLVM's solution: `SmallVector<T, N>` derives from `SmallVectorImpl<T>`, which derives from `SmallVectorBase`. The N parameter only controls the size of the inline buffer in the most-derived class. Methods are defined on `SmallVectorImpl<T>` and operate without knowing N. Function signatures take the base reference, the inline buffer is allocated wherever the most-derived value lives, and there is exactly one instantiation of the method code per element type.

## The default-N heuristic

`SmallVector<T>` (no explicit N) defaults to whatever inline count keeps `sizeof(SmallVector<T>)` near 64 bytes — one cache line. For `int` that's about 8–10 elements; for `std::string` it's 1–2. The choice trades inline capacity for object size; the 64-byte target reflects LLVM's empirical sweet spot from years of profiling its own data structures.

## When to use it

LLVM's design wins when you have:

- A large codebase with many functions taking SmallVector by reference.
- Heterogeneous N values across call sites.
- Sensitivity to binary size and instruction-cache pressure.

For a single project that picks one N and uses it everywhere, [[absl-inlinedvector|`absl::InlinedVector`]] is a closer match to `std::vector`'s API. For minimum sizeof, [[ankerl-svector]] beats both. For billions-of-vectors workloads with extreme memory pressure, [[folly-small-vector|`folly::small_vector`]] does. See [[small-buffer-optimization]] for the full SBO landscape and [[fastest-dynamic-arrays]] for where SBO sits relative to plain `std::vector` plus a good allocator.
