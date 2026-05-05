---
tags:
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# push_back_unchecked

`push_back_unchecked` is a non-standard vector operation that skips the `size == capacity` check, transferring responsibility for ensuring space to the caller. The performance impact is dramatically larger than the obvious "save one branch" intuition suggests: up to **5.9× speedup** at small sizes on Clang because removing the branch lets the compiler fully vectorize the loop. At 10M elements the speedup drops to 10–30%, but the technique remains a clean win in any context where the caller knows the upper bound.

## Why removing one branch is worth 5.9×

Inside a tight push_back loop, the compiler is constantly checking `if (size_ == capacity_)`. That branch:

- Carries a data dependency on the size field, blocking instruction-level parallelism.
- Has unpredictable behavior at the moment of growth — the predictor cannot know which iteration overflows.
- Inhibits autovectorization. The compiler refuses to emit SIMD stores when each iteration can either store or call a growth function.

When `push_back_unchecked` removes the check, the loop becomes a simple memory write per iteration. Clang autovectorizes it to AVX2/AVX-512 stores. The compiler can hoist the size pointer increment out of the loop body and apply it once at the end. The 5.9× number is the realistic impact of this transformation on small-to-medium sizes (where the branch dominates) — in 1000-int benchmarks: 540 ns → 93 ns.

## The MSVC penalty

Research from The Coding Nest measured MSVC's `std::vector` running **30–80% slower than a hand-rolled reimplementation** even in release builds. The penalty seems to come from a combination of debug-iterator support that survives release optimization, conservative inlining, and exception-safety bookkeeping that other implementations elide. A `push_back_unchecked` shim around MSVC's vector recovers most of the gap.

## How to use it safely

The standard pattern:

```cpp
v.reserve(known_upper_bound);
for (...) {
    v.push_back_unchecked(value);  // safe: we reserved enough
}
```

The caller's `reserve` makes the unchecked path correct. Bugs creep in when the upper bound is computed incorrectly — at which point you get a buffer overflow, not a graceful resize. This is one of those "safe by construction in the right context, unsafe by construction in the wrong context" patterns; it belongs in tightly-scoped hot loops, not as a default operation.

## Standard library availability

`std::vector` does not provide `push_back_unchecked`. [[folly-fbvector]] and [[absl-inlinedvector]] do, sometimes spelled `unchecked_emplace_back` or similar. Rust's `Vec` exposes `set_len` plus raw pointer writes (`as_mut_ptr().add(i)`), which together build the equivalent. Boost's containers expose it as `unchecked_push_back`.

## Where it fits

`push_back_unchecked` is the smallest of the [[fastest-dynamic-arrays|five major dynamic-array optimizations]] in absolute terms — `realloc`/[[realloc-mremap|mremap]] and good allocators dominate it. But it is the easiest to apply in a hot loop where you already know the size. For tight inner loops over bounded inputs, it is essentially free performance.

See [[fastest-dynamic-arrays]] for the broader hierarchy of optimizations.
