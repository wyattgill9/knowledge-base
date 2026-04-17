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

# frunk

frunk brings Haskell/Scala-style generic programming to Rust, inspired by Scala's shapeless library. It provides three major abstractions: heterogeneous lists, type-level sum types, and struct-to-struct conversion — all type-checked at compile time with zero runtime cost.

## HList (heterogeneous lists)

`HList!(i32, String, bool)` creates a type-safe heterogeneous list with type-safe indexing, mapping, folding, and appending. Unlike tuples, HLists support generic operations over their structure — you can write code that works on any HList regardless of length.

## Coproduct (type-level sum types)

`Coprod!(i32, f32, bool)` creates a type-level sum type with `inject`, `get`, and `fold` operations. This is the "advanced layer" alternative to [[either]]'s simple `Either<L, R>` — supporting any number of types with compile-time type checking.

## Generic / LabelledGeneric

The `Generic` trait converts structs to/from HLists, enabling **struct-to-struct conversion between structurally identical types** without manual field mapping. `LabelledGeneric` adds field-name awareness, so conversion works even when field order differs (as long as names and types match). This is frunk's killer feature for reducing boilerplate in DTO/model conversion layers.

## Validated (error accumulation)

`Validated` collects all errors from a computation rather than short-circuiting on the first one — the `Applicative`-style pattern from functional programming. Useful for form validation, configuration parsing, and anywhere you want to report all problems at once.

## When to use frunk

frunk is the right tool when you need type-level generic programming: struct conversion, heterogeneous collections, or type-safe coproducts. It's not a casual dependency — the learning curve mirrors Scala's shapeless. For simpler needs, [[either]] covers two-variant sum types, and [[derive-more]] handles basic trait derivation. See [[rust-enum-crates]] for the full landscape and [[typenum]] for the foundational type-level arithmetic.
