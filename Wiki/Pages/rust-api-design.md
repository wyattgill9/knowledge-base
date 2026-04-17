---
tags:
  - rust
  - design-patterns
  - architecture
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Rust API Design

The Rust API Guidelines establish the overarching philosophy: **accept the broadest reasonable input types, return the most specific output types.** This principle, applied consistently, makes APIs feel inevitable — callers rarely need explicit conversions, and return values carry maximum information.

## Accept broad, return specific

Never accept `&String` when `&str` works, or `&Vec<T>` when `&[T]` works. Rust's `Deref` coercion handles the conversion automatically, so accepting the borrowed form is strictly more flexible:

```rust
// Average: requires exact PathBuf
fn read_config(path: PathBuf) -> Config { /* ... */ }

// A+: accepts &str, String, PathBuf, &Path — anything path-like
fn read_config(path: impl AsRef<Path>) -> Config {
    let path = path.as_ref();
    // ...
}
```

For owned conversions, accept `impl Into<String>` rather than `String` — callers can pass `&str` without explicit `.to_string()`.

## The de-generification pattern

Monomorphization compiles a separate copy of each generic function for every concrete type used. The de-generification pattern keeps the generic wrapper thin and delegates to a concrete inner function compiled once:

```rust
pub fn open<P: AsRef<Path>>(path: P) -> Result<File> {
    open_impl(path.as_ref())  // Thin generic wrapper
}

fn open_impl(path: &Path) -> Result<File> {
    // Actual implementation — compiled once, not per-type
}
```

This prevents binary size bloat while maintaining ergonomic generic signatures. The standard library uses this pattern extensively.

## Three tiers of builders

1. **Mutable reference builder** — returns `&mut Self` for reusable configuration. `std::process::Command` uses this pattern. Simplest, but the builder is reusable after `.build()`.
2. **Consuming builder** — takes `self`, returns `Self`. Fluent chaining, single-use. Good when reconfiguring after build doesn't make sense.
3. **[[typestate-pattern|Typestate builder]]** — generic parameters track which required fields are set. `.build()` only exists when all required fields are `Set<T>`. Incomplete construction is a compile error. [[typed-builder]] and [[bon]] automate this with derive macros.

## Extension traits

Extension traits add methods to types you don't own. The convention (RFC 445): name them `FooExt`, implement via blanket `impl<T: BaseTrait> ExtTrait for T`, and export through a prelude module. Real-world examples: [[itertools]]'s `Itertools`, `futures::StreamExt`, `byteorder::WriteBytesExt`.

When defining traits, provide blanket implementations for `&T`, `&mut T`, and `Box<T>` where method signatures permit — this ensures trait objects and references work seamlessly.

## Visibility and forward compatibility

Expert crates are deliberate about their public surface. Internal modules use `pub(crate)` and `pub(super)`. Re-exports create a curated facade. Structs keep fields private — adding a field to an all-public struct is a breaking change; adding one to a struct with private fields is not. See [[facade-crate-pattern]] for how major crates organize this at workspace scale.

## Related patterns

See [[expert-rust-design]] for the full picture, [[type-driven-development]] for compile-time invariant techniques, and [[rust-abstraction-boundaries]] for when to use generics vs concrete types.
