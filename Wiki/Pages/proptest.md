---
tags:
  - rust
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# proptest

proptest is a property-based testing framework for Rust that generates shrinking counterexamples automatically. Unlike fuzzing ([[bolero]], cargo-fuzz), proptest is designed for logical correctness testing — "for all inputs satisfying X, property Y holds." See [[rust-build-tooling]].
