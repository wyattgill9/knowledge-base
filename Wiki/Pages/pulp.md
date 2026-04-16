---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# pulp

pulp provides portable SIMD abstractions for stable Rust with **built-in runtime CPU feature detection and dispatch**. A single binary runs optimally on SSE4.2, AVX2, AVX-512, and NEON without compile-time target selection.

## Why pulp over std::simd

`std::simd` remains nightly-only with stabilization blocked by API design issues. pulp works on stable Rust today and handles the runtime dispatch that `std::simd` would still require separately. For simpler use cases with fixed-width types and no runtime dispatch, [[wide]] is a lighter alternative.

## The stable SIMD landscape

As of Rust 1.87+, most `std::arch` SIMD intrinsics are now safe functions, making manual intrinsics more ergonomic. But pulp's value is the **dispatch layer** — writing one implementation that automatically selects the best instruction set at runtime.
