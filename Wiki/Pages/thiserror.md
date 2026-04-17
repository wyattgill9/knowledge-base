---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md"
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# thiserror

thiserror is the Rust ecosystem's standard derive macro for defining custom error types, with **~858M all-time downloads**. Its design goal is low conceptual overhead: you write normal enums/structs, annotate them, and the generated code is equivalent to handwritten `std::error::Error` and `Display` implementations. Version 2.0, released November 2024, added `no_std` support (leveraging `Error` moving to `core` in Rust 1.81 — see [[modern-rust-features]]), DST handling, and improved trait bound inference.

## Why thiserror won

The crate **deliberately does not appear in your public API**. The generated code is standard trait impls — switching between handwritten impls and thiserror is not a breaking change and not a semver hazard. This makes it uniquely attractive for libraries: downstream consumers are never forced to depend on thiserror.

The API is minimal: derive `Error`, annotate variants with `#[error("...")]` and `#[from]`/`#[source]`, done — roughly **2 lines per variant vs 5 for [[snafu]]**. Adopted by cargo, ruff, uv, tauri, and virtually every major Rust project.

```rust
use thiserror::Error;
use std::io;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("failed to read config file: {path}")]
    Read { path: String, #[source] source: io::Error },

    #[error("invalid config: {0}")]
    Parse(String),
}
```

## Backtrace support

thiserror's `#[backtrace]` attribute for capturing/providing backtraces requires a nightly compiler. In practice, teams pair thiserror with a top-level reporter ([[anyhow]], [[eyre]], [[miette]]) that captures backtraces on stable. This is the standard two-layer pattern: thiserror defines the types, a reporter handles presentation and backtraces.

## When Snafu is still better

[[snafu]]'s context selectors enforce more granular error types by design — each variant gets a generated companion type that attaches structured context at the call site. In large multi-crate workspaces, this forced discipline prevents error type degradation over time. For smaller projects, application-level errors, or maximum ecosystem familiarity, thiserror's simplicity wins.

## Migrating to Snafu

If you outgrow thiserror's model — typically because you want structured context at call sites rather than just variant definitions — the migration is mechanical: replace `#[derive(Error)]` + `#[error("...")]` with `#[derive(Snafu)]` + `#[snafu(display("..."))]`, then replace `map_err(|e| Error::Variant { source: e, ... })` with `.context(VariantSnafu { ... })`. Use `#[snafu(transparent)]` for transparent wrappers. See [[snafu-tutorial]] for the full migration guide.

## Cross-workspace error chaining

thiserror's `#[from]` attribute enables ergonomic error composition across workspace crates — see [[rust-workspace-patterns]] for the full pattern:

```rust
#[derive(thiserror::Error, Debug)]
pub enum ApiError {
    #[error("database error: {0}")]
    Database(#[from] my_db::DbError),   // auto-converts via ?
    #[error("config error: {0}")]
    Config(#[from] my_core::ConfigError),
}
```

## Companion crates

thiserror pairs naturally with:
- [[anyhow]] or [[eyre]] for application-level type erasure
- [[miette]] for diagnostic metadata (derive both `Error` and `Diagnostic` on the same type)
- [[displaydoc]] for doc-comment-driven display strings

See [[rust-error-handling]] for the full landscape and [[rust-error-crate-comparison]] for the detailed comparison.
