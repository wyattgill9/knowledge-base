---
tags:
  - rust
  - type-theory
  - design-patterns
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
last_updated: 2026-04-16
---

# Parse, Don't Validate

"Parse, don't validate" is a design principle articulated by Alexis King in November 2019 that captures the difference between two approaches to data checking:

- **A validator** checks data and throws away what it learned. It returns `bool` or `Result<(), Error>` — the caller knows the check passed but the type system doesn't.
- **A parser** checks data and returns a **more precise type** preserving that knowledge. Downstream code works with the refined type, never needing to re-check.

This is the philosophical foundation of [[type-driven-development]] and the principle that makes the [[newtype-pattern]] powerful.

## In Rust

```rust
// Validating: the type system forgets what we learned
fn is_valid_email(s: &str) -> bool { s.contains('@') }

// Parsing: the type system remembers
pub struct Email(String);
impl Email {
    pub fn parse(raw: String) -> Result<Self, &'static str> {
        if raw.contains('@') { Ok(Email(raw)) } else { Err("invalid email") }
    }
}
```

With the parsing approach, any function accepting `Email` is guaranteed to receive a validated value. The check happens once, at the boundary, and the type carries the proof forward.

## The boundary principle

Parse at system boundaries — user input, API responses, file reads, deserialization. Once data crosses the boundary into a refined type, internal code operates on types that are correct by construction. This eliminates:

- Redundant validation scattered through the codebase
- "Shotgun parsing" bugs where some paths check and others don't
- Documentation that says "this string must be a valid email" but the type system doesn't enforce it

## Relationship to other patterns

- **[[newtype-pattern]]** — the implementation mechanism (wrap + private constructor + validated `new`)
- **[[typestate-pattern]]** — the same principle applied to state transitions rather than data
- **Exhaustive matching** — the compiler verifying that all cases are handled (see [[rust-pattern-matching]])
- **[[nutype]]** — automates the boilerplate of creating validated newtypes

See [[type-driven-development]] for the full methodology.
