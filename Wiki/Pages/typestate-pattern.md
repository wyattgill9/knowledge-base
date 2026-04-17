---
tags:
  - rust
  - type-theory
  - design-patterns
sources:
  - "Raw/Rust/Type Driven Development in Rust.md"
last_updated: 2026-04-16
---

# Typestate Pattern

The typestate pattern is the crown jewel of [[type-driven-development]] in Rust. Each state in a state machine becomes a distinct type (or type parameter), and transitions are methods that **consume `self`** and return a new type. Rust's move semantics enforce that the old state is destroyed after a transition, making use-after-transition a compile error — no runtime checks needed, zero cost.

## The pattern

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;
struct Authenticated;

struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

impl Connection<Disconnected> {
    fn new(addr: &str) -> Self {
        Connection { address: addr.to_string(), _state: PhantomData }
    }
    fn connect(self) -> Connection<Connected> {
        Connection { address: self.address, _state: PhantomData }
    }
}

impl Connection<Connected> {
    fn authenticate(self, token: &str) -> Connection<Authenticated> {
        Connection { address: self.address, _state: PhantomData }
    }
}

impl Connection<Authenticated> {
    fn request(&self, path: &str) -> String {
        format!("GET {} from {}", path, self.address)
    }
}
```

- Calling `request` on `Connection<Disconnected>` — **compile error**
- Calling `connect` twice — **compile error** (first call consumed `self`)
- Using a connection after authentication — the only valid path

As Cliff Biffle wrote: "this pattern is so easy in Rust that it's almost obvious" — move semantics do the heavy lifting.

## Why Rust makes this easy

Other languages must simulate typestate with runtime checks or dynamic typing. Rust's three properties make it zero-cost:

1. **Move semantics** — consuming `self` destroys the old state, preventing use-after-transition
2. **`PhantomData<State>`** — zero-sized type parameter that carries state information with no memory or runtime overhead
3. **`impl` blocks parameterized by state** — methods exist only for specific states, so invalid operations don't even appear in the API

## Real-world usage

- **Serde's `Serializer` trait** — `serialize_struct` consumes the serializer and returns a `SerializeStruct` type. Calling `end()` consumes it and produces a `Result`. Invalid sequencing won't compile.
- **`imap-codec` crate** — typestate design made IMAP parsing injection-proof by construction
- **Embedded Rust** — hardware state machines where runtime errors are catastrophic

## Typestate builders

Combine with the builder pattern to ensure `.build()` is only callable when all required fields are set:

```rust
struct Missing;
struct Set<T>(T);

struct UserBuilder<Id, Email> { id: Id, email: Email, name: Option<String> }

impl UserBuilder<Set<String>, Set<String>> {
    fn build(self) -> User {
        User { id: self.id.0, email: self.email.0, name: self.name }
    }
}
// build() only exists when BOTH required fields are Set<String>
```

[[typed-builder]] and [[bon]] automate this with derive macros.

## When typestates break down

When state transitions depend on **runtime data** (user input, network events), the next state isn't known at compile time. You must fall back to enums with runtime dispatch or trait objects. The pragmatic heuristic from [[type-driven-development]]: encode structural invariants in types, validate value invariants at runtime boundaries.

## Sealed states

Combine with [[sealed-traits]] to prevent external code from defining new states, keeping the state machine closed and verifiable. See [[type-driven-development]] for the full methodology.
