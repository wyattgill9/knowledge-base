---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Fastest Dynamic Arrays in Computer Science

A C-style buffer using `realloc` on Linux — which transparently invokes [[realloc-mremap|mremap]] for large allocations — paired with [[mimalloc]] or [[jemalloc]] is the fastest dynamic array possible. It achieves O(1) growth without copying for large buffers and ~1.9 ns per append for trivially copyable types. Among production C++ containers, [[folly-fbvector]] with jemalloc is the fastest practical drop-in. The most important framing this work establishes is that **the allocator dominates the container** — a lesson that reorders almost every received opinion about vector design.

## The benchmark that reframes the field

On 100 million `uint64_t` push_back operations:

| Rank | Implementation | ns per insert |
|------|----------------|---------------|
| 1 | C `libdynamic` vector (glibc) | **1.92** |
| 2 | Rust `Vec<T>` (glibc) | **2.38** |
| 3 | C `libdynamic` vector ([[jemalloc]]) | 4.47 |
| 4 | Rust `Vec<T>` (jemalloc, default) | 4.91 |
| 5 | C++ `std::vector` (GCC 4.9) | 5.01 |
| 6 | LuaJIT vector assign | 6.82 |
| 7 | Java Trove `TLongArrayList` | 14.6 |
| 8 | Java `ArrayList<Long>` | 190 |

Switching allocators caused a larger swing than switching languages. The corollary is non-obvious: [[boxing-overhead|Java's ArrayList<Integer>]] is roughly 40× slower than C++ `std::vector` for primitive push_back not because the *container* is slow but because boxing taxes every element ~28–36 bytes. C# `List<T>` avoids this entirely through reified generics. Python's CPython interpreter compounds boxing overhead with ~50–100× interpreter cost. Once you control for the allocator and per-element cost structure, C++, Rust, and C land within ~10% of each other.

## Why std::vector cannot win

The fundamental ceiling on `std::vector` performance is the C++ allocation model. `new`/`delete` has no `realloc` equivalent, so growth always means: allocate fresh buffer, move-construct N elements, destroy originals, free old buffer. [[realloc-mremap|`realloc` with mremap]] avoids the copy entirely by remapping virtual pages — O(1) regardless of buffer size. [[jemalloc|jemalloc's `xallocx()`]] avoids the copy when adjacent memory is free. C++'s `std::allocator` interface exposes neither capability.

[[folly-fbvector]] sidesteps this by being explicitly jemalloc-aware: it queries `goodMallocSize()` to round requests to size classes (eliminating allocation slack), calls `xallocx()` to attempt in-place expansion, and treats [[trivial-relocatability|relocatable types]] as `memcpy`-compatible during reallocation. Facebook reported that applying the same philosophy to `fbstring` produced a 1% performance gain across their *entire* C++ codebase — staggering at that scale.

## The five techniques that actually move the needle

Ranked by measured effect:

1. **[[realloc-mremap|`realloc`/`mremap` for in-place growth]].** O(1) reallocation regardless of size. Tested: completely avoided copies from 0 to 256MB. Available only to C-style or C-allocator-aware vectors.
2. **High-performance allocators.** [[mimalloc]] delivers 13–22% speedup vs system malloc with zero code changes. [[jemalloc]] gives 30% higher throughput at 36 threads. [[tcmalloc]] hits 50× system libmalloc throughput on 4MB allocations. Linking is the single highest-leverage change available.
3. **SIMD memcpy.** AVX2/AVX-512 copying achieves 2–20× scalar speed on large aligned blocks. Modern `memcpy` already SIMD-vectorizes, but explicit alignment via `xsimd::aligned_allocator` ensures the fastest paths. See [[simd-programming]].
4. **[[trivial-relocatability]].** Using `memcpy` instead of element-wise move during reallocation: 2–10× speedup on the realloc itself. Folly's `IsRelocatable` and the P1144 proposal expose what types can opt in. Most of the standard library qualifies — `std::string`, `std::unique_ptr`, every standard container — even though `std::is_trivially_copyable` says otherwise.
5. **[[push-back-unchecked]].** Eliminating the capacity branch when the caller guarantees space: up to 5.9× speedup at small sizes on Clang because the loop fully vectorizes. Even at 10M elements: 10–30%. MSVC's `std::vector` runs 30–80% slower than a hand-rolled reimplementation in release.

## Small buffer optimization is overrated

[[small-buffer-optimization|SBO]] vectors embed a fixed inline buffer to skip allocation for small sizes. The four major C++ designs make different tradeoffs: [[llvm-smallvector|LLVM's SmallVector]] uniquely offers `SmallVectorImpl<T>` type erasure to prevent template bloat; [[absl-inlinedvector|Google's `absl::InlinedVector`]] aims for perfect `std::vector` API compatibility; [[folly-small-vector|`folly::small_vector`]] packs the inline/heap flag into a single bit for billions-of-vectors workloads; [[ankerl-svector]] hits a remarkable 8-byte minimum sizeof.

But the empirical results are unflattering. SBO pays a branch on every access (inline vs. heap). For 1000-int push_back, plain `std::vector` actually wins. Random access on `absl::InlinedVector` was measured ~5× slower than `std::vector`. Rust's [[smallvec|`SmallVec`]] is up to 5.5× slower than `Vec` with reserved capacity for `from_slice`. The benchmark author concluded malloc is fast enough that SBO complexity often hurts more than it helps.

The genuinely fixed-capacity case has a cleaner answer: [[static-vector|`boost::container::static_vector`]] (and the upcoming `std::static_vector` via P0843) eliminates even the inline/heap branch, giving true O(1) non-amortized push_back.

## Growth factor: a secondary concern

Languages disagree on the [[growth-factor-analysis|growth multiplier]]: GCC/Clang `std::vector` and Rust `Vec` use 2×; MSVC and Java use 1.5×; Python uses ~1.125×; Go adapts from 2× to ~1.25×. The classic argument that "1.5× allows reuse of freed memory under first-fit allocators while 2× cannot" is correct, but the amortized speed difference is small. For large workloads, fbvector's adaptive 2× → 1.5× transition is the most defensible choice.

## Theory beats practice on paper, loses in benchmarks

Brodnik, Carlsson, Demaine, Munro, and Sedgewick (1999) proved a lower bound: any dynamic array maintaining O(1) amortized operations must waste at least Ω(√N) space. Tarjan and Zwick (SOSA 2023) sharpened this to a formal space-time tradeoff: arrays using N + O(N^{1/r}) space must pay Ω(r) amortized grow cost. [[space-optimal-arrays|Hashed Array Trees and Tiered Vectors]] hit these bounds in O(√N) space.

Yet Katajainen's empirical work shows space-optimal structures are "measurably slower" than `std::vector`. The extra indirection level defeats hardware prefetchers and adds cache misses that more than erase the savings. This is a recurring lesson — see also [[cache-oblivious-structures]] and the broader [[fastest-data-structures]] survey.

## The hierarchy

For raw speed: a C buffer with `realloc` and mimalloc/jemalloc, accepting trivially-copyable elements only. For C++ projects: [[folly-fbvector]] with [[jemalloc]] — the production-grade ceiling. For everything else: `std::vector` or `Vec<T>` with a [[rust-memory-allocators|replacement allocator]] gets you within 2–3× of optimal with one line of code. SBO and `push_back_unchecked` are situational. Boxing and interpreter overhead in Java/Python dominate everything else and cannot be undone with container cleverness.
