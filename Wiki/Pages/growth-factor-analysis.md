---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Growth Factor Analysis

The growth factor — the multiplier by which a dynamic array's capacity expands when it overflows — is one of those design choices that generates outsized debate relative to its measurable impact. Major implementations have settled on three different values: 2× (GCC/Clang `std::vector`, Rust `Vec`), 1.5× (MSVC, Java), and ~1.125× (Python). Go takes a fourth option and adapts. The amortized speed difference between them is small. The memory-reuse argument is correct but secondary.

## The choices in production

| Implementation | Growth factor | Rationale |
|----------------|--------------|-----------|
| GCC/Clang `std::vector` | 2× | Simplicity, single-instruction shift |
| Rust `Vec<T>` | 2× | Aligns with C++; doubling matches small-allocation size classes |
| MSVC `std::vector` | 1.5× | Permits memory reuse under first-fit allocators |
| Java `ArrayList` | 1.5× | Same reuse argument |
| Python `list` | ~1.125× | Conservative; Python lists are pointer-heavy and grow slowly |
| Go slices | 2× → ~1.25× | Adaptive: doubles small slices, tapers for large ones |
| [[folly-fbvector]] | 2× → 1.5× | Adaptive: doubles small (size-class friendly), 1.5× large (reuse friendly) |

## The memory reuse argument

The classic case for 1.5× over 2×: with a strict first-fit allocator that places fresh allocations after the existing free list, **2× growth can never reuse previously freed memory**. Each new allocation is strictly larger than the sum of all prior ones (1 + 2 + 4 + ... + 2^k = 2^{k+1} - 1 < 2^{k+1}), so freed buffers can never be the right size for the next request. With a 1.5× factor, the geometric sum eventually exceeds the next requested size, and previously freed regions become candidates for reuse.

The argument is correct in the abstract. In practice, modern allocators ([[mimalloc]], [[jemalloc]], [[tcmalloc]]) maintain size-class free lists rather than purely first-fit free lists, so the geometric arithmetic is not what determines reuse. The dominant factor is whether the allocator's size class for the new request matches a class that has freed entries — which depends on the allocator's class spacing, not the container's growth factor.

## Why the speed difference is small

Amortized push_back cost is `(growth_factor / (growth_factor - 1)) * (cost of one element copy)`. For 2× growth that is 2 copies per push amortized; for 1.5×, 3 copies; for 1.125×, 9 copies. Growth happens log_g(N) times, so total work scales identically — only the constant factor changes. With [[realloc-mremap|`realloc`/mremap]] avoiding most copies anyway, the constant factor difference shrinks toward zero.

The peak memory waste differs more dramatically: 2× wastes up to 50% of allocated memory after a recent grow; 1.5× wastes 33%; 1.125× wastes 11%. For memory-constrained systems this matters; for throughput-critical systems running on machines with abundant RAM, it usually does not.

## The adaptive choice is the most defensible

Both Go and [[folly-fbvector]] transition from doubling at small sizes to slower growth at large sizes. The reasoning compounds two arguments: small allocations match jemalloc/mimalloc size classes that are themselves powers of two (so doubling is "free" — you would have been rounded up anyway), while large allocations dominate memory consumption (so 1.5× materially reduces waste). It is the right answer if you only get one growth factor.

## What actually matters more

The growth factor is a third-order optimization. The first-order optimization is the [[rust-memory-allocators|allocator]]. The second-order optimization is [[realloc-mremap|in-place expansion]] (which decouples growth cost from buffer size, making the growth factor argument about *waste*, not about *speed*). Once those are handled, growth factor becomes a memory-vs-speed tuning knob with predictable behavior in either direction. See [[fastest-dynamic-arrays]] for the full hierarchy.
