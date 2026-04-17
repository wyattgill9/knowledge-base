---
tags:
  - rust
  - performance
  - design-patterns
  - crate
sources:
  - "Raw/Rust/Rust crates that supercharge enums and algebraic data types.md"
last_updated: 2026-04-16
---

# enum_dispatch

enum_dispatch converts trait-object-style polymorphism (`Box<dyn Trait>`) into static enum-based dispatch, achieving **up to 10x speedup** by eliminating vtable lookups. Annotate a trait and enum with `#[enum_dispatch]`, and it auto-generates match-based delegation plus `From`/`TryInto` conversions for each variant.

## How it works

```rust
#[enum_dispatch]
trait Animal {
    fn name(&self) -> &str;
}

#[enum_dispatch(Animal)]
enum MyAnimal {
    Dog(Dog),
    Cat(Cat),
}
```

The macro generates a `match` arm for each variant, calling the trait method on the inner type directly. No heap allocation, no indirection, no vtable. It also enables serde serialization of "trait objects" — something impossible with `dyn Trait`.

## Tradeoffs

- **Same-crate constraint**: trait and enum must be defined in the same crate
- **Poor IDE support**: rust-analyzer and RustRover struggle with the generated code
- **No associated types**: limited support for complex trait signatures

## Alternatives

| Crate | Advantage over enum_dispatch |
|---|---|
| `enum_delegate` | Cross-crate support, better error messages, associated types |
| `declarative_enum_dispatch` | Perfect IDE support via declarative macros |
| [[ambassador]] | Most general: structs + enums, cross-crate, generics |
| `delegate` | Method-level granularity with parameter modifiers (22.7M+) |

The right choice depends on your constraints: enum_dispatch for maximum performance in a single crate, [[ambassador]] for cross-crate generality, `declarative_enum_dispatch` when IDE support matters. See [[rust-enum-crates]] for the full dispatch landscape.
