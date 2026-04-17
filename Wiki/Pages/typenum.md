---
tags:
  - rust
  - type-theory
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# typenum

typenum provides type-level numbers evaluated at compile time, with full arithmetic (`Add`, `Sub`, `Mul`, `Div`, `Pow`), comparisons, and bitwise operations. Its `op!` macro enables ergonomic expressions like `op!(U3 + U4)`. It is the foundation of type-level programming in Rust, underpinning [[generic-array]], `dimensioned` (compile-time SI/CGS unit checking), and much of the RustCrypto ecosystem.

## Why typenum still matters

Rust's const generics can handle basic `[T; N]` arrays, but they cannot yet do arithmetic in associated type positions — you can't express `Array<N> + Array<M> = Array<N+M>` with const generics alone. typenum fills this gap with trait-based arithmetic that works today on stable Rust. [[generic-array]] bridges the two worlds, and `hybrid-array` provides a modern transition path.

## Related crates

- **[[generic-array]]** — arrays generic over their length via typenum
- **`dimensioned`** — compile-time unit checking using typenum for exponents
- **`static_assertions`** — compile-time assertions about type equality, trait implementation, size, and alignment
- **[[frunk]]** — uses typenum-style concepts for HList indexing and type-level computation
- **`tyrade`** — an embedded functional language that compiles type-level enums and functions into traits/impls, with syntax like Peano arithmetic

See [[rust-enum-crates]] for the full type-level programming landscape.
