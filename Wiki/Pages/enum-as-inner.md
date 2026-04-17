---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# enum_as_inner

enum_as_inner derives `as_*()`, `as_*_mut()`, `into_*()`, and `is_*()` accessor methods for each enum variant, returning `Option`. This eliminates full `match` expressions when you just need to extract one variant's data.

Its fork `enum_try_as_inner` returns `Result` instead, providing rich error info (expected vs actual variant plus original value recovery) — invaluable for state machines. [[derive-more]] also derives `IsVariant`, `Unwrap`, and `TryUnwrap` as a broader alternative. See [[rust-enum-crates]].
