---
tags:
  - rust
  - type-theory
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# generic-array

generic-array provides arrays generic over their length via [[typenum]], still essential because Rust's const generics cannot yet do arithmetic in associated type positions. It underpins much of the RustCrypto ecosystem and any code needing type-level array length manipulation.

`hybrid-array` bridges const generics and typenum for a modern transition path. See [[rust-enum-crates]] and [[typenum]].
