---
tags:
  - rust
  - crate
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Itertools

itertools is the canonical extension trait crate for Rust's `Iterator` trait, providing methods like `chunks`, `tuple_windows`, `interleave`, `join`, and dozens more. It follows the [[rust-api-design|extension trait]] convention (RFC 445): the `Itertools` trait is blanket-implemented for all `Iterator` types and re-exported via a prelude.
