---
tags:
  - rust
  - architecture
sources:
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Modern Rust Features

A comprehensive guide to Rust language evolution from 2023–2026, covering the 2024 edition, async revolution, type system completion, pattern matching ergonomics, stdlib additions, and project structure patterns. Rust between 1.75 and 1.94 gained async closures, native async traits, trait upcasting, let-chains, inline const blocks, and dozens of ergonomic improvements across 20 releases.

## The three most impactful changes

1. **Native async fn in traits** (1.75) — replaced the `async-trait` proc-macro's heap allocations with zero-cost compiler-generated futures. See [[rust-async-evolution]].
2. **The 2024 edition** (1.85) — changed lifetime capture rules, scoping semantics, unsafe ergonomics, and added async closures as first-class citizens. See [[rust-2024-edition]].
3. **Let-chains** (1.88) — eliminated deeply nested `if let` pyramids. See [[rust-pattern-matching]].

## Feature timeline

| Version | Date | Key features |
|---|---|---|
| 1.65 | Nov 2022 | [[gats]], [[let-else]] |
| 1.75 | Dec 2023 | [[rust-async-evolution\|Async fn in traits]], RPITIT |
| 1.77 | Mar 2024 | C-string literals (`c"hello"`) |
| 1.78 | May 2024 | `#[diagnostic::on_unimplemented]` |
| 1.79 | Jun 2024 | Inline `const {}` blocks |
| 1.80 | Jul 2024 | [[lazy-lock\|LazyLock/LazyCell]], exclusive range patterns |
| 1.81 | Sep 2024 | `Error` in `core`, `#[expect(lint)]` |
| 1.82 | Oct 2024 | `&raw` operator, precise capture `use<>`, `min_exhaustive_patterns` |
| 1.85 | Feb 2025 | **2024 edition**, async closures (`AsyncFn`), resolver "3" |
| 1.86 | Apr 2025 | [[trait-upcasting]], `get_disjoint_mut` |
| 1.88 | Jun 2025 | [[rust-pattern-matching\|Let-chains]] (2024 edition only) |
| 1.89 | Jul 2025 | Inferred const generics (`[u8; _]`) |

## What's still missing

- **TAIT** (Type Alias Impl Trait) — blocked on the next-generation trait solver
- **The never type** (`!`) — also blocked on the new solver
- **Pin ergonomics** — making pinning less painful
- **Async iterators via `gen` blocks** — `gen` keyword reserved in 2024 edition
- **Pattern types** and **try blocks** — expected for the 2027 edition

## Key pages

- [[rust-2024-edition]] — edition-specific changes
- [[rust-async-evolution]] — async fn in traits, async closures, structured concurrency
- [[gats]] — Generic Associated Types
- [[rust-pattern-matching]] — let-else, let-chains, exhaustive patterns
- [[rust-workspace-patterns]] — virtual workspaces, inheritance, feature flags
- [[trait-upcasting]] — `&dyn Sub` to `&dyn Super` coercion
- [[lazy-lock]] — `LazyLock`/`LazyCell` replacing `lazy_static`/`once_cell`
