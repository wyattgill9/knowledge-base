---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# enum-map

enum-map provides `EnumMap<K, V>` — a map keyed by enum variants backed by a plain array. It guarantees **O(1) access** with no hashing and exhaustive initialization (every variant must have a value), making it ideal when every variant needs an associated value. Think of it as a type-safe, zero-overhead lookup table. See [[rust-enum-crates]].
