---
tags:
  - cpp
  - design-patterns
  - architecture
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Modern C++ Design Patterns (C++23 and Beyond)

The classical Gang-of-Four catalog assumed a language where polymorphism meant virtual functions, error handling meant exceptions or status codes, and abstraction always cost an indirection. C++23 and C++26 dismantle most of those assumptions: explicit object parameters retire CRTP, `std::expected` retires the exception/error-code dilemma, `std::generator` retires hand-rolled iterators, and senders/receivers retire ad-hoc thread pools. The unifying philosophy is **value semantics, compile-time composition, and zero runtime cost** — patterns now express themselves through the type system rather than through inheritance hierarchies.

## The ten patterns

The source distills ten patterns that displace older idioms. Each one is a small page on its own; the table below is the map.

| Pattern | Replaces | Key feature | Standard |
|---|---|---|---|
| [[deducing-this]] | [[crtp\|CRTP]] | `this auto&&` explicit object parameter | C++23 |
| [[std-expected]] | exceptions / error codes | monadic `and_then` / `transform` / `or_else` | C++23 |
| [[std-generator]] | hand-rolled iterators | `co_yield` coroutines composing with ranges | C++23 |
| [[std-mdspan]] | matrix classes, raw pointer math | non-owning multidim view + layout policies | C++23 |
| [[if-consteval]] | `std::is_constant_evaluated()` hacks | compile-time/runtime strategy in one body | C++23 |
| [[std-move-only-function]] | inheritance for callable storage | move-only type-erased polymorphism | C++23 |
| [[cpp26-static-reflection]] | macros, codegen, protobuf-for-trivia | `^T`, splicers, `template for` | C++26 |
| [[senders-receivers]] | thread pools, callback chains | `std::execution` structured concurrency | C++26 |
| [[overloaded-visit-pattern]] | virtual dispatch (closed sets) | `std::visit` + variadic overload set | C++17, refined |
| [[cpp26-contracts]] | `assert()` macros | language-level `pre` / `post` clauses | C++26 |

## The three philosophical shifts

**From inheritance to value semantics.** [[deducing-this]] makes the base class non-template again; [[std-move-only-function]] stores any callable as a value; [[overloaded-visit-pattern]] dispatches on a `std::variant` with compile-time exhaustiveness instead of a `virtual` table. The hierarchy of `Shape → Circle → Square` that defined OOP-era C++ is now a closed `std::variant<Circle, Square, ...>`, and the compiler will catch a missing case the day you add `Triangle`.

**From runtime to compile time.** [[if-consteval]] lets a single function carry both implementations and pick at parse time. [[cpp26-static-reflection]] generates JSON serialization, ORM mappers, and RPC stubs directly from struct definitions — replacing the macro-and-codegen layer that defined every serialization library since the 2000s. [[cpp26-contracts]] hoists `assert()` into the language so the compiler, not the preprocessor, decides whether to check.

**From "throw or return -1" to monadic flow.** [[std-expected]] makes a fallible call a value, and `.and_then(...)` / `.transform(...)` / `.or_else(...)` compose a pipeline that short-circuits on the first error. This is C++'s direct adoption of [[railway-oriented-programming]] and the same `Result<T, E>` discipline that defines idiomatic Rust — see [[rust-error-handling]] for the Rust side of the convergence.

## Why C++26 matters more than C++23

C++23 is mostly refinement: better iterators, better error type, better view over memory. C++26 is the larger leap. **[[cpp26-static-reflection|Static reflection]]** finally gives C++ what every "modern" language (Java annotations, C# attributes, Rust derive macros) had for decades — the ability to ask the compiler about types and generate code from the answer. **[[senders-receivers|`std::execution`]]** introduces structured concurrency with cancellation propagation, scheduler abstraction, and composable pipelines, displacing the patchwork of `std::async`, raw `std::thread`, executor-library forks, and callback-chained futures. **[[cpp26-contracts|Contracts]]** add a Design-by-Contract syntax that the optimizer can see, so `pre(size > 0)` can simultaneously document, check in debug, and inform code generation in release.

Together they pull C++ much closer to the rest of the type-driven, structured-concurrency, derive-driven world that Rust, Swift, and modern Kotlin already inhabit.

## The unspoken pattern: convergence with Rust

Read the patterns end-to-end and the convergence with Rust is unmistakable. [[std-expected|`std::expected<T, E>`]] is `Result<T, E>`. [[std-generator|`std::generator<T>`]] is a Rust generator block. [[std-move-only-function|`std::move_only_function`]] is `Box<dyn FnOnce>`. [[overloaded-visit-pattern]] is `match` on an `enum`. [[cpp26-contracts|Contracts]] echo the typestate-and-newtype discipline of [[type-driven-development]]. The two languages are not merging, but they are converging on the same answers — value semantics, sum types, monadic error flow, structured concurrency, compile-time metaprogramming — because those answers are correct.

The C++ side carries one durable advantage: zero runtime overhead is the default rather than the option. The Rust side carries its own: the borrow checker enforces what C++ leaves to discipline. The patterns above are the C++ side of the bargain — they assume the discipline; C++26 contracts and reflection are tightening the assumption.

## What this source doesn't cover

The source is a pattern catalog, not a survey. It deliberately skips ranges-and-views beyond a brief `std::generator` example, modules (still painful in 2026 despite ~3 years of shipping), the executor proposals around GPU dispatch, and the long-running Safe C++ / Profiles debate. Several of those will need their own pages as additional sources arrive — flagged in the open-questions section of the ingest log.
