---
tags:
  - rust
  - data-structures
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# derive_more

derive_more is the Swiss-army knife of Rust derive macros, with **200M+ all-time downloads**. It generates trait implementations for enums, structs, and newtypes across four categories: conversions, formatting, error handling, and operators. It is particularly valuable for the newtype pattern and enums wrapping inner types.

## Conversions

- `From`, `Into`, `TryFrom`, `TryInto` — automatic conversion impls between wrapper types and their inner types
- `FromStr` — parse strings into types
- `IntoIterator` — make types iterable

## Display and formatting

- `Display`, `Binary`, `Octal`, `LowerHex`, `UpperHex` — format types with configurable templates

## Error handling

- `Error` — generates `std::error::Error` implementations with automatic `source` and backtrace detection from field types and names. Supports `no_std` except for `provide()` (which requires `Backtrace` from `std`). A viable alternative to [[thiserror]] when you want one derive crate for multiple traits rather than a dedicated error-handling crate. For structured context beyond basic derives, [[snafu]] or [[error-stack]] remain the better choices. See [[rust-error-crate-comparison]].

## Operators

- Arithmetic: `Add`, `Sub`, `Mul`, `Div`, `Rem`
- Unary: `Not`, `Neg`
- Assign: `AddAssign`, `SubAssign`, etc.
- Index: `Index`, `IndexMut`
- Deref: `Deref`, `DerefMut`

## Utility methods

- `Constructor` — generate `new()` methods
- `IsVariant` — `is_variant()` methods for enums
- `Unwrap` — `unwrap_variant()` methods (panics on wrong variant)
- `TryUnwrap` — `try_unwrap_variant()` returning `Result`

The `IsVariant`, `Unwrap`, and `TryUnwrap` derives overlap with [[enum-as-inner]]'s accessor methods but are bundled into a single dependency.

## When to use derive_more

Use derive_more when you need trait derivation across multiple categories — conversions, formatting, operators — from a single crate. For enum-specific ergonomics (iteration, string conversion, metadata), [[strum]] is more focused. For error types specifically, [[thiserror]] is the ecosystem standard with wider adoption. See [[rust-enum-crates]] for the full landscape.
