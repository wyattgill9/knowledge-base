---
tags:
  - rust
  - type-theory
  - crate
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
last_updated: 2026-04-16
---

# typed-builder

typed-builder is a Rust derive macro that generates builder types with compile-time required field checking. It automates the typestate builder pattern from [[type-driven-development]]: `.build()` only exists as a method when all required fields have been set, producing a compile error if any are missing.

The **`bon`** crate extends this concept to function arguments, providing the same compile-time guarantees for function call sites. Both crates add proc-macro compile overhead but eliminate the manual boilerplate of typestate builders (one type per field-state combination). See [[typestate-pattern]] for the underlying mechanism.
