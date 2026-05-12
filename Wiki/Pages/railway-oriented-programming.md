---
tags:
  - error-handling
  - design-patterns
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Railway-Oriented Programming

Railway-Oriented Programming (ROP) is Scott Wlaschin's metaphor for monadic error handling: model every function as a piece of two-track railway — a success track and a failure track — and compose them with operators that automatically switch onto the failure track on the first error and bypass all subsequent steps. The success track is `Result.map` / `transform`; the failure track is short-circuiting. Wlaschin coined the metaphor in 2014 talks targeting F# developers; the pattern itself is the `Either` / `Result` monad that Haskell adopted in the 1990s. In 2026 it is the dominant error-handling idiom across Rust ([[rust-error-handling|`Result<T, E>`]]), C++23 ([[std-expected|`std::expected<T, E>`]]), Swift (`Result<Success, Failure>`), Scala (`Either`), Kotlin (`kotlin.Result`), and most modern functional languages.

## The metaphor

Imagine a railway with two parallel tracks. Each function is a switch on the railway: input arrives on the success track, the function either passes it through (staying on success) or routes it onto the failure track. Once on the failure track, no further function processes the value — the failure simply rolls forward to the end of the pipeline.

```
input ─[parse]─[validate]─[normalize]─[save]─ result
                                                  
  ╲___ failure track ____________________________╱
```

Translated to types:

- Every function returns `Result<T, E>` (or `Expected<T, E>`).
- `map(f)` / `transform(f)` operates on the success track, leaving the failure track alone.
- `and_then(f)` / `flat_map(f)` operates on the success track but `f` can itself return a failure, switching onto the failure track.
- `or_else(f)` operates on the failure track — useful for recovery or alternative paths.

## The C++ / Rust translation

The clearest demonstration is identical-shape code in both languages:

```rust
// Rust
let result = parse_int(input)
    .and_then(to_celsius)
    .map(|c| format!("{:.1}°C", c))
    .or_else(|e| { log_error(e); Err(e) });
```

```cpp
// C++23 — same pipeline, slightly different vocabulary
auto result = parse_int(input)
    .and_then(to_celsius)
    .transform([](double c) { return std::format("{:.1f}°C", c); })
    .or_else([](ParseError e) -> std::expected<std::string, ParseError> {
        log_error(e); return std::unexpected(e);
    });
```

The shapes are identical because the underlying algebra is identical: `Result<T, E>` / [[std-expected|`std::expected<T, E>`]] is the same monad with the same laws.

## What it replaces

**`try` / `catch` ladders.** ROP makes error propagation explicit in the type while keeping the body linear. Exceptions can do this too but are invisible in the signature and impose runtime costs.

**`if (err) return err;` ladders.** Manual error propagation works but is verbose and breaks reading flow. ROP makes the propagation a property of the operator rather than a statement at every step.

**Deeply nested conditionals on error tags.** When every function returns a `(value, error_code)` pair, callers nest `if (!err1) { ... if (!err2) { ... } }` indefinitely. ROP flattens this into a sequence of pipeline steps with the failure track handling the bookkeeping.

## The trap nobody mentions

ROP pipelines are elegant on the page but obscure stack traces. When a Rust pipeline of five `and_then` steps fails on the third, the resulting error has no record of which step failed — only the error value itself. Production-grade Rust solved this with [[snafu]], [[error-stack]], and [[anyhow]]-style context attachment: each step in the pipeline attaches a "context note" so the final error carries a trail.

C++ has not yet solved this. `std::expected` ships without context-attachment idioms; the third-party ecosystem is still consolidating. This is the open frontier in C++ error handling as of 2026 — see [[std-expected]] and [[rust-error-handling]] for both sides of the trade-off.

## Theoretical grounding

ROP is a friendly name for the **`Either` monad** (Haskell's term) or the **`Result` monad** (Rust's term) — `Either e a` for fixed error type `e` and varying success type `a`, with `return a = Right a`, `(>>=) :: Either e a -> (a -> Either e b) -> Either e b`. The monad laws (left identity, right identity, associativity) hold for `Result` and `expected` and underwrite the legitimate refactorings you can do to a pipeline without changing semantics.

The category-theoretic framing doesn't matter for using ROP, but it matters for understanding why the pattern feels right: it's the simplest non-trivial monad after `Maybe` / `Option`, the one where the failure carries information, and it composes with sequencing the way exceptions do but with explicit types throughout.

## Beyond Wlaschin

Wlaschin's contribution wasn't inventing the monad — it was making it accessible to non-functional-language developers via the railway metaphor and translating the academic terminology (monad, bind, fmap) into the operational vocabulary (railway, switch, two tracks) that production engineers absorbed immediately. The talk and book *Domain Modeling Made Functional* (2018) extended the metaphor into a full methodology for type-driven domain modeling that connects directly to [[type-driven-development]] in Rust and the [[modern-cpp-design-patterns]] convergence in C++.

The pattern's broader significance: it is the single design idea that has propagated furthest from "functional programming" into mainstream language design over the past decade. C++23, Rust, Swift, Kotlin, modern Java (`Optional` chains), and TypeScript (`Result` libraries) all converged on it independently from the same starting point — that exceptions and error codes are both wrong, and that errors should be values with composition rules.
