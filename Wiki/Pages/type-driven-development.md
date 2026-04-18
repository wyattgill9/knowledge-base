---
tags:
  - rust
  - type-theory
  - design-patterns
  - architecture
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-17
---

# Type Driven Development

Type Driven Development (TDD) is a methodology where types come first — serving as specifications, design documents, and proof obligations — so the compiler becomes an active collaborator that rejects invalid programs before they ever run. In Rust, move semantics, algebraic data types, and zero-cost abstractions let you encode state machines, business rules, and domain invariants directly into the type system with no runtime penalty.

The methodology traces its name to Edwin Brady's 2017 book on the Idris language ("Type, define, refine"), but its practical expression in Rust draws on three foundational slogans:

- **"Make illegal states unrepresentable"** — Yaron Minsky (Jane Street, 2010). Structure algebraic data types so invalid data combinations cannot exist.
- **[[parse-dont-validate|"Parse, don't validate"]]** — Alexis King (2019). A validator checks data and throws away what it learned. A parser checks data and returns a more precise type preserving that knowledge.
- **"If it compiles, it works"** — the Rust community's informal maxim, reflecting that a well-designed type system catches large classes of bugs at compile time.

## The seven techniques

### 1. The newtype pattern

Wrap primitives in single-field tuple structs to create distinct, incompatible types. Zero-cost — the compiler erases the wrapper. Combined with private constructors, this becomes [[parse-dont-validate]] in action. See [[newtype-pattern]].

### 2. Phantom types and zero-sized types

`PhantomData<T>` carries type-level information (units, currencies, states) at zero runtime cost. No memory, no instructions, no overhead. This is the mechanism that makes the [[typestate-pattern]] possible.

### 3. The typestate pattern

The crown jewel. Each state becomes a distinct type; transitions consume `self` and return a new type. Rust's move semantics enforce that the old state is destroyed — use-after-transition is a compile error. See [[typestate-pattern]].

### 4. Making illegal states unrepresentable

Model each valid state as an enum variant carrying only relevant data, instead of using `Option` fields that permit meaningless combinations:

```rust
// Bad: allows both None — 4 representable states, 3 valid
struct Contact { email: Option<String>, phone: Option<String> }

// Good: representable states = valid states
enum ContactInfo {
    Email(String),
    Phone(String),
    Both { email: String, phone: String },
}
```

Pablo Mansanet's concept of **"tightness"** formalizes this: a type is 100% tight when every possible value it can hold is semantically valid.

### 5. Exhaustive pattern matching as verification

When you add a new variant, the compiler flags every `match` that must handle it. No forgotten cases, no runtime surprises. This is the compiler acting as a completeness checker for domain logic. See [[rust-pattern-matching]] for the modern syntax.

### 6. Sealed traits for closed type sets

A public trait requiring a private supertrait creates a closed set of types that external code cannot extend. See [[sealed-traits]].

### 7. Typestate builders

Builder pattern combined with typestates ensures `.build()` is only callable when all required fields are set. [[typed-builder]] and [[bon]] automate this with derive macros. See [[typestate-pattern]] for the underlying mechanism.

## The payoff

- **Compile-time bug elimination** — entire categories (null dereferences, invalid transitions, forgotten cases, argument mixups) cannot occur
- **Self-documenting code** — `fn transfer(from: Account<Active>, to: Account<Active>, amount: PositiveAmount) -> Result<Receipt, InsufficientFunds>` communicates its contract without documentation
- **Refactoring confidence** — change a domain model, the compiler flags every affected location
- **Zero runtime cost** — PhantomData compiles away, newtypes are representationally identical, typestate transitions are just function calls. Type-level machinery generates no additional instructions

## Where it breaks down

**Verbosity** — each state needs its own struct, generic builders accumulate type parameters, newtypes need trait derives. Crates like [[nutype]], [[typed-builder]], and `typestate` automate boilerplate but add proc-macro compile overhead.

**Compile times** — heavy type-level programming causes exponential growth. [[typenum]]'s authors created `tnfilt` specifically to make error messages readable.

**Not all invariants are encodable** — subtracting two `StrictlyPositive` values doesn't guarantee non-zero. String formats, numeric ranges on computed values, and complex interdependent business rules resist compile-time encoding.

**Dynamic environments** — when state transitions depend on runtime data (user input, network events), the [[typestate-pattern]] breaks down because the next state isn't known at compile time. The Krustlet team had to fall back to trait objects.

**Rust is not dependently typed** — unlike Idris, Rust cannot prove algorithmic correctness. It focuses on memory safety, thread safety, and structural correctness.

## The pragmatic heuristic

Encode **structural** invariants (state machines, required fields, protocol compliance) in types. Validate **value** invariants (string formats, numeric ranges) at runtime boundaries. Resist the temptation to push every single rule into the type system.

## Essential resources

- Edwin Brady, *Type-Driven Development with Idris* (Manning, 2017)
- Will Crichton, *Type-Driven API Design in Rust* (online book + Strange Loop 2021)
- Cliff Biffle, "The Typestate Pattern in Rust" (cliffle.com, 2019)
- Alexis King, "Parse, Don't Validate" (2019)
- Scott Wlaschin, *Domain Modeling Made Functional* (2018)
- Luca Palmieri, *Zero to Production in Rust* (domain invariants chapter)

See [[expert-rust-design]] for how these techniques manifest across every dimension of production Rust — API surface design, crate architecture, error handling, and performance.
