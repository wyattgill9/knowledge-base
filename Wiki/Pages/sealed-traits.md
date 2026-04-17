---
tags:
  - rust
  - type-theory
  - design-patterns
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
last_updated: 2026-04-16
---

# Sealed Traits

The sealed trait pattern creates a closed set of types that external code cannot extend. A public trait requires implementing a private supertrait, so only types within your crate can satisfy the bound:

```rust
mod private { pub trait Sealed {} }

pub trait ValidState: private::Sealed { /* ... */ }

struct Active;
struct Suspended;

impl private::Sealed for Active {}
impl private::Sealed for Suspended {}
impl ValidState for Active {}
impl ValidState for Suspended {}

// External code cannot create new states — the set is closed
```

This is essential for the [[typestate-pattern]] when you want to guarantee that only your defined states exist. Combined with `From` and `TryFrom` for type-safe conversions, sealed traits let you build compile-time verified domain models where the valid state space is exactly what you define. See [[type-driven-development]] for the full methodology.
