---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# num_enum

num_enum handles safe enum-to-integer and integer-to-enum conversions via `IntoPrimitive`, `TryFromPrimitive`, and `FromPrimitive` derives. It supports `#[num_enum(alternatives = [...])]` for multiple integer values mapping to one variant and `#[num_enum(catch_all)]` for wildcard variants.

This is critical for **FFI and protocol parsing** where raw `as` casting silently truncates or produces UB. [[strum]]'s `FromRepr` covers simpler cases; num_enum is the better choice when you need alternatives, catch-all, or `TryFrom` semantics. See [[rust-enum-crates]].
