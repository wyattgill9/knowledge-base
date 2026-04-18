---
tags:
  - rust
  - type-theory
  - design-patterns
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-17
---

# Newtype Pattern

The newtype pattern wraps primitive types in single-field tuple structs, creating distinct types that the compiler treats as incompatible. A function `fn register(username: Username, email: Email)` makes argument swaps impossible, unlike `fn register(username: String, email: String)`. Newtypes are a **zero-cost abstraction** — the compiler optimizes away the wrapper entirely.

## Parse, don't validate

The pattern becomes most powerful with **private constructors and validated creation** — this is [[parse-dont-validate]] in action:

```rust
mod domain {
    pub struct Email(String);

    impl Email {
        pub fn new(raw: String) -> Result<Self, &'static str> {
            if raw.contains('@') { Ok(Email(raw)) }
            else { Err("invalid email address") }
        }
    }
}
// Email("bad".into()) won't compile — the constructor is private
// Once you hold an Email, you know it passed validation
```

Downstream code never needs to re-check. The `TryFrom` trait integrates this into Rust's conversion ecosystem.

## Reducing boilerplate

The main cost is implementing `Display`, `Debug`, `Clone`, `PartialEq`, and other traits for each wrapper. Solutions:

- **[[derive-more]]** — derives `From`, `Into`, `Display`, `Deref`, and more for newtypes
- **[[nutype]]** — proc macro that generates validated newtypes with sanitization rules automatically (2.7M+ downloads)
- **`Deref` / `AsRef`** — provide transparent access to the inner type when appropriate. **Critical rule**: do *not* implement `Deref` for safety newtypes — if the newtype exists to restrict an API, `Deref` would bypass the restriction entirely. `Deref` is appropriate only for smart pointer-like wrappers that extend capabilities while preserving the inner type's full surface

## When to use newtypes

Use newtypes when **two values of the same primitive type have different semantic meaning** — user IDs vs order IDs, meters vs kilometers, raw strings vs validated strings. The compile-time cost is zero. The cognitive cost is minimal. The bug prevention is permanent.

See [[type-driven-development]] for the full methodology and [[typestate-pattern]] for the related technique using phantom types.
