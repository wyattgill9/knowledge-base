---
tags:
  - rust
  - data-structures
  - design-patterns
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# Recursion Schemes

The `recursion` crate implements proper recursion schemes — catamorphisms (recursive collapse) and anamorphisms (recursive expand) — in Rust with **guaranteed stack safety** on arbitrarily deep trees. It is underused relative to its power; anyone building AST-heavy code should consider it over hand-rolled traversals.

## Core abstractions

The crate separates recursion machinery from logic, just as iterators separate iteration from logic:

- **`MappableFrame`** — the functor: defines how to map over one layer of a recursive structure
- **`Collapsible`** — recursive collapse (catamorphism): fold a tree bottom-up into a value
- **`Expandable`** — recursive expand (anamorphism): unfold a value into a tree

Under the hood, it uses **topologically-sorted arena-based traversal** for cache-friendly performance. This means no stack overflow on deep trees, and better cache locality than naive recursive descent.

## Why this matters

Hand-rolled tree traversals interleave recursion mechanics with transformation logic, making them hard to compose, test, and verify for stack safety. Recursion schemes factor these concerns apart: you write the "what to do at each node" logic, and the scheme handles traversal order, stack management, and memory layout. This is the same insight that makes `Iterator` so powerful — but applied to trees.

## Related tools

- **`derive-visitor`** — derivable Visitor/Drive pairs for enum-based ASTs (simpler but less powerful)
- **`syn` visit/fold** — gold standard for Rust syntax tree traversal specifically
- **`fix_fn`** — Y combinator for ad-hoc recursive closures (simpler, not stack-safe)
- **`visita`** — visitor with compile-time exhaustiveness checking

See [[rust-enum-crates]] for the full visitor/recursion landscape.
