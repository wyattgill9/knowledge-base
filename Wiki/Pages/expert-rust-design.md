---
tags:
  - rust
  - architecture
  - design-patterns
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Expert Rust Design

A distillation of what separates A+ Rust from average implementations, drawn from the Rust API Guidelines, production patterns at OneSignal, RisingWave, and ShakaCode, and architectural analysis of [[tokio]], [[serde-architecture|serde]], [[axum-architecture|axum]], and [[bevy]]. The unifying principle: **push invariants from runtime to compile time**.

## The expert mental model

Experts treat borrow checker errors as **design feedback**, not obstacles. When the compiler rejects code, it means the data ownership model has a flaw worth fixing — not a puzzle to trick the compiler into accepting with `clone()` or `Rc<RefCell<T>>`.

The entire memory model decomposes along two orthogonal axes: **ownership** (responsibility for cleanup) and **access** (shared `&T` or exclusive `&mut T`). From these two concepts, every rule derives: `Send` means safe to transfer ownership across threads, `Sync` means safe to share access, and `&T is Send` iff `T is Sync`.

The function signature hierarchy: need to destroy it? Take `T`. Need to modify it? `&mut T`. Just reading? `&T`. Small `Copy` type? Let Rust copy it. Beginners default to cloning; experts restructure data flow.

**Structs own all their data; methods borrow parameters.** Storing references in structs (`&'a T`) cascades lifetime complexity through the codebase. The expert alternative: restructure so the struct owns its data, borrows appear only in function parameters.

## Core techniques

The seven compile-time techniques from [[type-driven-development]] form the foundation: [[newtype-pattern|newtypes]], phantom types, [[typestate-pattern|typestate]], illegal-state elimination, exhaustive matching, [[sealed-traits]], and typestate builders. But A+ design extends beyond individual patterns into API surface design, crate architecture, and knowing when to stop abstracting.

## Dimensions of expert design

- **[[rust-api-design]]** — accept broad, return specific; de-generification; three tiers of builders; extension traits
- **[[serde-architecture]]** / **[[axum-architecture]]** — how expert crates use traits for zero-cost extensibility
- **[[facade-crate-pattern]]** — workspace of specialized crates behind a curated public API
- **[[rust-allocation-patterns]]** — borrow → Cow → owned hierarchy; niche optimization; buffer reuse
- **[[rust-concurrency-patterns]]** — Arc<Mutex> as last resort; data-oriented state splitting; async task patterns
- **[[rust-abstraction-boundaries]]** — enums vs generics vs trait objects; macros as last resort; the overengineering trap
- **[[rust-testing-patterns]]** — layered testing; doc tests with `?`; CI lint enforcement

## The deepest insight

The crates that define Rust's ecosystem — serde, tokio, axum, bevy — share architectural DNA: [[facade-crate-pattern|facade crates]] over modular workspaces, trait-driven extensibility, derive macros for ergonomics, and feature flags for compile-time modularity. The highest-quality Rust code is not the cleverest — it's the code where the type system does the thinking, the compiler does the checking, and the developer does the designing.
