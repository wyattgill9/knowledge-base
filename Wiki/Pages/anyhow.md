---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md"
last_updated: 2026-04-16
---

# anyhow

anyhow provides `anyhow::Result<T>`, a type-erased error type optimized for Rust application code where you don't care about the returned error type — you just want to propagate failures with context and get useful reports. It is the fastest path to "good enough" application error handling.

## Core API

anyhow works with any type implementing `std::error::Error`. The key ergonomic tools:

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("reading config from {path}"))
}

fn main() -> Result<()> {
    let cfg = load_config("app.toml")?;
    println!("{cfg}");
    Ok(())
}
```

- `Context` / `with_context` — attach string context to any `Result`
- `bail!` — early return with a formatted error
- `anyhow!` — create an ad-hoc error

## Backtraces

Backtraces are supported on stable Rust and gated by environment variables (`RUST_BACKTRACE`, `RUST_LIB_BACKTRACE`). No nightly required.

## `no_std` support

anyhow supports `no_std` with the default `"std"` feature disabled, but requires a global allocator (heap usage and dynamic dispatch). This is a meaningful portability advantage over UI-heavy crates like [[miette]] or [[color-eyre]], though for truly allocation-free environments, hand-rolled error enums or [[snafu]] with careful feature tuning are better fits.

## Positioning: anyhow vs thiserror

anyhow's README is explicit: use anyhow when you *don't care* what error type your functions return (typical for binaries). Use [[thiserror]] when you *do care* (typical for libraries). This is the canonical two-layer pattern — [[thiserror]] or [[snafu]] at library boundaries, anyhow at the application boundary.

## Boundary conversion from SNAFU

[[snafu]] errors implement `std::error::Error + Send + Sync + 'static`, so `?` converts them into `anyhow::Error` automatically via `From`. This makes the layering pattern clean: use typed SNAFU errors with context selectors inside library/domain modules, then propagate into `anyhow::Result` at `main` or the service boundary. See [[snafu-tutorial]] for examples.

## Anyhow vs eyre

[[eyre]] is a fork of anyhow with swappable report handlers, giving you control over how errors are formatted. If you need custom presentation (colors, sections, SpanTraces via [[color-eyre]]), use eyre. If you just need error propagation with context, anyhow is simpler and has fewer dependencies.

## Anyhow vs error-stack

[[error-stack]] pushes toward structured, typed contexts and arbitrary attachments rather than string context. If you find yourself wishing anyhow's context strings were queryable or typed, error-stack is the upgrade path — at the cost of more development overhead.

See [[rust-error-handling]] for the full landscape and [[rust-error-crate-comparison]] for the detailed comparison.
