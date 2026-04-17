---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# bitflags

bitflags generates struct types that behave like C-style bitflag sets via the `bitflags!` macro. With **~1.1 billion all-time downloads**, it is one of the top three most-downloaded crates on crates.io and is used by the Rust compiler itself.

## What it provides

- Bitwise operations (`|`, `&`, `^`, `!`) on flag sets
- Iteration over set flags
- String parsing and formatting
- `const`-compatible construction
- Exhaustive flag definitions with named constants

## Alternatives

**`enumflags2`** uses real `enum` syntax with a `#[bitflags]` attribute instead of a macro, providing a type-level distinction between a single flag (`YourEnum`) and a flag set (`BitFlags<YourEnum>`). **`enumset`** creates compact bitset-backed sets with set algebra operations (union, intersection, difference). Both are viable but bitflags' dominance makes it the default. See [[rust-enum-crates]].
