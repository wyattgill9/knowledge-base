---
tags:
  - rust
  - data-structures
  - design-patterns
  - type-theory
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# Rust Enum Crates

A comprehensive survey of 60+ crates that extend Rust's enum system — from eliminating daily boilerplate to enabling type-level programming rivaling Haskell. The ecosystem splits into a pragmatic layer (table-stakes for most projects) and an advanced layer (compile-time guarantees most languages can't express).

## The pragmatic layer

These crates have hundreds of millions of downloads each and solve daily friction:

- **[[strum]]** (100M+) — the standard for enum ergonomics: `Display`, `FromStr`, `EnumIter`, `EnumCount`, `EnumProperty`, `EnumDiscriminants`, `FromRepr`. One crate replaces five.
- **[[bitflags]]** (~1.1B) — C-style bitflag sets via `bitflags!` macro. Used by the Rust compiler itself.
- **[[derive-more]]** (200M+) — Swiss-army knife: derives `From`, `Into`, `Display`, `Error`, operator overloading, and utility methods for enums and newtypes.
- **[[thiserror]]** (~858M) — the standard for error type derives. See [[rust-error-handling]].
- **[[enum-dispatch]]** — converts `dyn Trait` into static match-based delegation for up to 10x speedup over vtable lookups.

## Category 1: Enum utilities

| Crate | Role |
|---|---|
| [[strum]] | Iteration, Display, FromStr, metadata, discriminants |
| [[bitflags]] | Bitwise flag sets with const construction |
| `enumflags2` | Real enum syntax for bitflags with type-distinct flag vs flag-set |
| `enumset` | Compact bitset-backed sets with set algebra |
| [[enum-map]] | O(1) array-backed map keyed by enum variants |
| `enum-iterator` | Rich `Sequence` trait with cyclic navigation |
| `enum-assoc` | Declarative data/function attachment per variant |
| `variant_counter` | Analytics: track variant occurrences with statistical functions |

## Category 2: Pattern matching ergonomics

| Crate | Role |
|---|---|
| [[enum-as-inner]] | Derives `as_*()`, `into_*()`, `is_*()` accessor methods returning `Option` |
| `enum_try_as_inner` | Fork returning `Result` with error info and value recovery |
| `enum-kinds` / `kinded` | Generate companion fieldless "kind" enums |
| `if_chain` | Flatten nested `if let` chains (partially superseded by Rust 1.88+ let-chains) |
| `guard` | Swift-style `guard let` for ergonomic early returns |
| `nestum` | Readable patterns for nested enum hierarchies |
| `vesta` | Extensible pattern matching for plugin architectures |

## Category 3: Derive macros

| Crate | Role |
|---|---|
| [[derive-more]] | From, Into, Display, Error, operators, utility methods |
| [[thiserror]] | Error types with Display, source chains, backtrace |
| [[num-enum]] | Safe integer-to-enum conversions with alternatives and catch-all |
| [[auto-enums]] | Auto-generate dispatch enums for different-typed branches |
| `spire_enum` | Hygienic enum delegation with variant type tables |
| `strum-lite` | Lightweight strum alternative using declarative macros |

## Category 4: Open/extensible enums

| Crate | Role |
|---|---|
| `open-enum` | Newtype struct with associated constants; accepts unknown integer values (FFI) |
| `anon_enum` | Pre-built `Enum2` through `Enum16` for ad-hoc sum types |
| `typeunion` | TypeScript-inspired `type MyType = A + B + C` with subtype relationships |
| `enumx` | Ad-hoc enum types with enum exchange and checked exceptions |

## Category 5: Visitor pattern and recursion schemes

| Crate | Role |
|---|---|
| `derive-visitor` | Derivable Visitor/Drive pairs with enter/exit events |
| `visita` | Visitor with compile-time exhaustiveness checking |
| `syn` visit/fold | Gold standard for Rust AST traversal |
| `synstructure` | Utilities for derive macro authors |
| [[recursion-schemes]] | Proper catamorphisms/anamorphisms with stack safety |
| `fix_fn` | Y combinator for recursive closures |

## Category 6: Enum dispatch (replacing dyn Trait)

| Crate | Tradeoff |
|---|---|
| [[enum-dispatch]] | Canonical, 10x over `Box<dyn Trait>`, but same-crate only, poor IDE support |
| `enum_delegate` | Cross-crate, better errors, handles associated types |
| `declarative_enum_dispatch` | Perfect IDE support via declarative macros |
| [[ambassador]] | Most general: structs and enums, cross-crate, generics (2.3M+) |
| `delegate` | Method-level granularity with parameter modifiers (22.7M+) |
| `auto-delegate` | Route different traits to different struct fields |

## Category 7: Generic sum types

| Crate | Approach |
|---|---|
| [[either]] | Canonical `Either<L, R>` with trait delegation |
| [[frunk]] `Coproduct` | Type-level sum types with inject/get/fold |
| `choice` | Recursive choice with automatic trait composition |
| `summum-types` | Full-featured sum type macro with method dispatch |
| `trait-union` | Stack-allocated trait objects from fixed implementor sets |

## Category 8: Type-level programming

| Crate | Capability |
|---|---|
| [[typenum]] | Type-level numbers with full arithmetic (foundation for RustCrypto, generic-array) |
| [[generic-array]] | Arrays generic over length via typenum |
| [[frunk]] | HList, Coproduct, Generic/LabelledGeneric, Validated |
| `tyrade` | Embedded functional language for type-level computation |
| `higher-kinded-types` | `ForLifetime` trait for `<'_>`-genericity on stable |
| `refl` | Type equality witnesses for safe casting |
| `session_types` | Communication protocols as types |
| `tlist` | Type-level linked lists via GATs |
| `dimensioned` | Compile-time unit checking (SI, CGS) |
| `static_assertions` | Compile-time assertions about types, traits, sizes |

## Three ecosystem gaps worth watching

1. **Anonymous sum types** — `anon_enum`, `typeunion`, `auto_enums` work around the absence of a native `A | B | C` type. An RFC could eventually land.
2. **Recursion schemes adoption** — the [[recursion-schemes|recursion]] crate is underused relative to its power. Anyone building AST-heavy code should consider it.
3. **Dispatch crate competition** — [[enum-dispatch]] vs [[ambassador]] vs `delegate` form a competitive space. Choice depends on cross-crate support, IDE compatibility, or method-level granularity needs.
