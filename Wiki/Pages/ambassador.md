---
tags:
  - rust
  - design-patterns
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# ambassador

ambassador is the most general-purpose Rust trait delegation crate (**2.3M+ downloads**). It delegates trait implementations via `#[delegatable_trait]` and `#[derive(Delegate)]`, working for both structs (delegating to fields) and enums (delegating to variants), with cross-crate and generic support.

Compared to [[enum-dispatch]] (which is same-crate only with poor IDE support), ambassador trades a small amount of macro complexity for significantly broader applicability. For method-level delegation rather than whole-trait delegation, `delegate` (22.7M+) offers finer control with parameter modifiers like `#[into]`, `#[as_ref]`, and `#[newtype]`. See [[rust-enum-crates]] for the full dispatch landscape.
