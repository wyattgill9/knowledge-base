---
tags:
  - rust
  - design-patterns
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# auto_enums

auto_enums solves a common Rust pain point: when different branches of a function return different concrete types implementing the same trait. Annotate with `#[auto_enum(Iterator)]` and it generates a hidden enum that delegates the trait, avoiding `Box<dyn Iterator>`.

Companion crates `iter-enum` and `io-enum` extend this to specific trait families. For full enum-based trait dispatch, see [[enum-dispatch]] and [[ambassador]]. See [[rust-enum-crates]] for the full landscape.
