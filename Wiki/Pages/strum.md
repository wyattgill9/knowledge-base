---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# strum

strum is the single most popular Rust enum ergonomics crate, with **100M+ all-time downloads**. It provides derive macros that cover the most common enum boilerplate: `Display`, `FromStr`, iteration, counting, metadata, discriminant extraction, and integer conversion — what would otherwise require five separate dependencies.

## Key derives

- **`Display`** / **`AsRefStr`** — convert variants to strings with configurable casing and formatting
- **`EnumString`** (`FromStr`) — parse strings into enum variants
- **`EnumIter`** — iterate all variants of a fieldless or unit-variant enum
- **`EnumCount`** — `const COUNT: usize` for the number of variants
- **`EnumProperty`** — attach custom key-value metadata per variant via `#[strum(props(key = "value"))]`
- **`EnumDiscriminants`** — generate a separate fieldless "kind" enum mirroring the data-carrying one
- **`EnumIs`** — `is_variant()` methods for each variant
- **`FromRepr`** — convert integers to enum variants (safer than `as` casting)

## When to use strum

strum is the default choice when you need any combination of enum-to-string, string-to-enum, or variant iteration. For error types specifically, [[thiserror]] or [[snafu]] are better fits. For broader trait derivation beyond enums (From, Into, operators), [[derive-more]] covers more ground. For bitflag-style enums, [[bitflags]] is the standard.

`enum-iterator` offers a richer iteration API than strum's `EnumIter` (cyclic navigation, `next()`/`previous()`, works on composite types), but strum's bundled approach is usually sufficient. `strum-lite` provides a lighter alternative using declarative macros for faster compilation.

See [[rust-enum-crates]] for the full landscape.
