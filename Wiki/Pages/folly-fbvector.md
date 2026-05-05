---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# folly::fbvector

Facebook's `folly::fbvector` is the fastest production-quality drop-in replacement for `std::vector`. It is API-compatible by construction, but four compounding design choices give it a meaningful edge: [[jemalloc]]-aware allocation sizing, in-place expansion via `xallocx()`, [[trivial-relocatability]] using `memcpy`, and an adaptive growth factor that doubles for small buffers and shifts to 1.5× for large ones. The cumulative effect is described by Folly's own documentation as "always non-negative, frequently significant, sometimes dramatic, and occasionally spectacular."

## The four advantages

**jemalloc-aware allocation sizing.** When fbvector detects jemalloc at compile time, it routes growth requests through `goodMallocSize()` to round them up to jemalloc's nearest size class. This eliminates allocation slack — the rounding the allocator will perform anyway becomes visible to the container, which can offer that capacity to the user instead of wasting it. `std::vector` cannot do this because `std::allocator` provides no interface for asking "what would you actually allocate if I asked for N bytes?"

**In-place expansion via `xallocx()`.** Reallocation in fbvector first attempts to grow the existing allocation in place. jemalloc's `xallocx()` returns the new effective size if the adjacent memory is free, allowing the buffer to grow with **zero copying**. Testing with 256M floats showed `xallocx` avoided copies for most growth attempts on buffers above 32KB. `std::vector` cannot perform this because C++'s `new`/`delete` model has no `realloc` equivalent — see [[realloc-mremap]] for why this is the single most powerful optimization for large arrays.

**Trivial relocatability.** Types annotated with `FOLLY_ASSUME_FBVECTOR_COMPATIBLE` are relocated using `memcpy` instead of element-wise move construction during reallocation. This extends beyond `std::is_trivially_copyable` — `std::unique_ptr<T>`, `std::string`, and most standard containers are safely bitwise-relocatable even though the C++ standard does not formally recognize this. The pending P1144 proposal would standardize the trait. See [[trivial-relocatability]] for the full mechanics.

**Adaptive growth strategy.** Small buffers double — exploiting the fact that jemalloc's small-allocation size classes pack 2× requests cleanly. Large buffers grow at 1.5×, enabling the freed memory of previous allocations to be reused by future ones. See [[growth-factor-analysis]] for why this matters less than people think but still beats either pure choice.

## Production scale validates the design

Facebook applied the same optimization philosophy to `fbstring` and reported a **1% performance improvement across their entire C++ codebase** — a staggering gain at company scale, justifying significant engineering investment in low-level container work. The `fbvector` document claims gains that "are always non-negative, frequently significant, sometimes dramatic, and occasionally spectacular."

## When to reach for it

Pick fbvector when (a) you control the global allocator and can link jemalloc, (b) your hot path involves vectors that grow over their lifetime, and (c) your element types are trivially relocatable or can be marked as such. Without jemalloc, fbvector falls back to standard allocation and most of its advantage evaporates — at which point [[abseil-flat-hash-map|Abseil]]-style containers or `std::vector` with a better allocator are equally good choices. See [[fastest-dynamic-arrays]] for the broader landscape.
