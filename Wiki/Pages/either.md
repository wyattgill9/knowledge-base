---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# either

either provides the canonical `Either<L, R>` sum type with `Left` and `Right` variants. It implements `Iterator`, `Read`, `Write`, and other standard traits via delegation to the inner type. Simple, battle-tested, and ubiquitous in the Rust ecosystem.

For more than two types, [[frunk]]'s `Coproduct` provides type-level sum types (`Coprod!(i32, f32, bool)`) with inject/get/fold operations, and `anon_enum` offers pre-built `Enum2` through `Enum16`. See [[rust-enum-crates]] for the full generic sum type landscape.
