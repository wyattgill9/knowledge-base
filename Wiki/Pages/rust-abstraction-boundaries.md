---
tags:
  - rust
  - architecture
  - design-patterns
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Rust Abstraction Boundaries

The most subtle expert skill is knowing **when not to abstract**. The choice between enums, generics, and trait objects is not a matter of preference — each has a specific domain.

## Enums vs generics vs trait objects

| Mechanism | Best for | Trade-off |
|-----------|----------|-----------|
| **Enums** | Small, closed, fixed variant sets (errors, AST nodes, messages) | Adding operations is easy (add a match arm); adding variants is breaking |
| **Generics** | Hot paths with compile-time-known types | Zero-cost monomorphization, but increases binary size |
| **Trait objects** | Plugin systems, heterogeneous collections, runtime-determined types | Open extensibility, but vtable dispatch (~5x slower in tight loops) |

The key insight: enums and trait objects solve opposite problems. Enums make adding operations easy but adding variants hard. Trait objects make adding types easy but adding methods hard. Choose based on which axis of extension matters.

## Macros as last resort

The decision flowchart: Can a function solve it? Use a function. Need variable argument counts or code generation? Consider a macro.

Functions are testable, debuggable, and type-checked. Macros generate code that is harder to debug and produces opaque error messages. Declarative `macro_rules!` suffices for simple repetitive patterns. Procedural macros (derive, attribute) are appropriate for code generation at the scale of [[serde-architecture|serde]]'s or Clap's derive APIs.

## The overengineering trap

The deepest trap is **premature generification**. Average Rust code that's trying too hard introduces trait bounds and generic parameters that add complexity without benefit. The expert instinct:

1. Write concrete code first
2. Identify the actual variation axis
3. Abstract along exactly that axis — never preemptively

A+ code uses concrete types until abstraction is actually needed, then introduces the **minimum viable abstraction**. Three similar lines of code is better than a premature trait hierarchy.

## Related pages

See [[expert-rust-design]] for the full design philosophy, [[rust-api-design]] for the de-generification pattern, and [[enum-dispatch]] for when static dispatch can replace trait objects at 10x speedup.
