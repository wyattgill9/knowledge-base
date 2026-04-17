---
tags:
  - rust
  - architecture
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
  - "Raw/Rust/Type Driven Development in Rust.md"
last_updated: 2026-04-16
---

# Rust Pattern Matching

Rust's pattern matching received major ergonomic improvements between 2022 and 2025: let-else for refutable binding, let-chains for flattening nested conditionals, exclusive range patterns, and exhaustive pattern simplification. Together they eliminate the majority of boilerplate in pattern-heavy code.

## let-else (Rust 1.65)

Refutable binding with mandatory divergence — the workhorse of modern early-return patterns:

```rust
fn parse_header(input: &str) -> Option<(u64, &str)> {
    let Some((count_str, name)) = input.split_once(':') else {
        return None;
    };
    let Ok(count) = count_str.parse::<u64>() else {
        return None;
    };
    Some((count, name.trim()))
}
```

The `else` block must diverge (`return`, `break`, `continue`, `panic!`). This replaces the common `if let Some(x) = ... { ... } else { return }` pattern with a flat, linear flow.

## Let-chains (Rust 1.88, 2024 edition only)

Chain multiple `let` patterns with `&&` in `if`/`while` — eliminating deeply nested `if let`:

```rust
// Requires edition = "2024"
fn process(headers: Option<&Headers>, body: Option<&Body>, authenticated: bool) {
    if let Some(h) = headers
        && h.content_type() == "application/json"
        && let Some(b) = body
        && b.len() > 0
        && authenticated
    {
        println!("Processing {} bytes of JSON", b.len());
    }
}

// Also works in while loops
fn drain_valid(iter: &mut impl Iterator<Item = Option<i32>>) {
    while let Some(opt) = iter.next()
        && let Some(val) = opt
        && val > 0
    {
        println!("Got: {val}");
    }
}
```

Let-chains partially supersede the `if_chain` crate (see [[rust-enum-crates]]), though `if_chain` remains useful on older toolchains.

## Exclusive range patterns (Rust 1.80)

Complete the range pattern syntax with exclusive upper bounds:

```rust
fn classify(value: u32) -> &'static str {
    match value {
        0 => "zero",
        1..10 => "single digit",    // exclusive end
        10..100 => "two digits",
        100..1000 => "three digits",
        _ => "large",
    }
}
```

## min_exhaustive_patterns (Rust 1.82)

Omit impossible arms when matching uninhabited types:

```rust
use std::convert::Infallible;

fn unwrap_infallible(x: Result<i32, Infallible>) -> i32 {
    let Ok(val) = x; // No Err arm needed — Infallible is uninhabited
    val
}
```

## Exhaustive matching as verification

Beyond syntax, Rust's exhaustive `match` is a powerful [[type-driven-development]] tool. When you add a new variant to a domain enum, the compiler flags every `match` that must handle it — acting as a **completeness checker** for domain logic. No forgotten case handlers, no runtime surprises. This is why "make illegal states unrepresentable" (see [[type-driven-development]]) pairs so well with exhaustive matching: the enum encodes only valid states, and `match` forces you to handle every one.

## Relationship to other features

These pattern matching improvements work best in combination: let-else for early returns, let-chains for multi-condition guards, and the [[rust-2024-edition]]'s tighter temporary scoping for safe lock handling in pattern contexts. For type-driven design patterns that leverage exhaustive matching, see [[type-driven-development]] and [[typestate-pattern]]. See [[modern-rust-features]] for the full timeline.
