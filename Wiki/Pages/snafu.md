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

# Snafu

Snafu is a Rust [[rust-error-handling|error-handling]] library that generates errors and adds structured information to underlying errors via *context selectors*. Its core insight is that the same underlying error type (e.g., `io::Error`) can occur in multiple conceptual situations, and each deserves a distinct, semantically named error variant with structured fields capturing *what the program was attempting*. It remains a strong choice for large multi-crate workspaces — GreptimeDB, InfluxDB IOx, and Iroh all use it — while [[thiserror]] 2.0 holds the position of ecosystem default for simpler projects. See [[snafu-tutorial]] for the full tutorial with progressive examples.

## Context selectors: the core abstraction

`#[derive(Snafu)]` generates a *context selector* type for each error variant — a companion struct that strips `source` and `backtrace` fields (SNAFU handles those automatically) and is used to construct errors via `.context()` and `.with_context()`:

```rust
use snafu::prelude::*;
use std::{fs, io, path::PathBuf};

#[derive(Debug, Snafu)]
pub enum Error {
    #[snafu(display("Could not open config at {}", path.display()))]
    OpenConfig { path: PathBuf, source: io::Error },

    #[snafu(display("Could not write output to {}", path.display()))]
    WriteOutput { path: PathBuf, source: io::Error },
}

pub fn load_then_write(config: &PathBuf, out: &PathBuf) -> Result<(), Error> {
    let bytes = fs::read(config).context(OpenConfigSnafu { path: config.clone() })?;
    fs::write(out, bytes).context(WriteOutputSnafu { path: out.clone() })?;
    Ok(())
}
```

A critical ergonomic detail: selectors call `Into::into` on each field, so you can pass `&str` into a `String` field without explicit conversion. A common beginner mistake is writing `.context(Error::Variant { ... })` — you must use the *selector* (`VariantSnafu`), not the enum path.

## Selector visibility and module-local design

Context selectors are **private by default**, reflecting SNAFU's opinion that modules should define cohesive error types locally. This reduces accidental coupling between modules. Use `#[snafu(visibility(pub))]` on the error type or specific variants when cross-module access is needed. Exporting selectors in a public API is warned against — their shape may change across SNAFU versions, creating semver hazards.

For public API stability, SNAFU documents an **opaque error type** pattern: a public newtype wrapping an internal SNAFU error enum, with delegated traits and a `From` conversion. This lets you use full SNAFU machinery internally while presenting a stable, minimal error surface to consumers.

## Semantic backtraces

Context selectors push codebases toward multiple small, meaningful error types at each architectural layer. The resulting error chain reads as a "semantic backtrace" — not which functions were on the call stack, but *what the program was attempting at each layer*. This is more debuggable than a raw backtrace, especially in async code where backtraces show executor frames.

## Progressive error design

SNAFU explicitly recommends a graduation path:

1. **`Whatever` + `whatever!`** — turnkey string errors for prototyping. Maximum speed, minimum structure.
2. **Typed enums with context selectors** — when you hit limitations or want structured context at call sites.
3. **Opaque newtype wrappers** — for public API stability in libraries.

## Key ergonomic tools

- **`context(SelectorSnafu { ... })`** — attach cheap context (fields already available)
- **`with_context(|err| SelectorSnafu { ... })`** — lazy context, evaluated only on error path; closure receives the source error for inspection
- **`OptionExt::context`** — converts `Option<T>` into `Result<T, E>` with structured context (when "missing value" is a domain error)
- **`ensure!(condition, SelectorSnafu { ... })`** — early return if condition fails
- **`SelectorSnafu.fail()`** — create a leaf error (no underlying source)
- **`#[snafu(transparent)]`** — delegate `Display` and `source()` to the underlying error
- **`#[snafu(context(false))]`** — allow bare `?` without a selector (use sparingly — context is usually what makes errors actionable)

## Backtrace patterns

SNAFU's `Backtrace` type is env-gated: capture is a no-op unless `RUST_BACKTRACE` or `RUST_LIB_BACKTRACE` is set.

- **Leaf errors**: include `backtrace: snafu::Backtrace` — zero cost when disabled.
- **Hot-path errors**: use `backtrace: Option<snafu::Backtrace>` — captures only when the user explicitly enables it. Critical for errors used as flow control.
- **Chained errors**: don't add backtrace fields to errors with a `source` — the leaf's backtrace propagates through the chain.

## Reporting

`Display` prints only the top-level message (SNAFU's default no longer includes the source's `Display`, aligning with broader Rust guidance). For full chain + backtrace output:

- **`#[snafu::report]`** on `main` — sugar over returning `Report`. Works with `tokio::main`, `async_std::main`, etc.
- **`snafu::Report`** — explicit report type for more control.
- **`CleanedErrorText`** — lower-level tool for deduplicating chain messages.
- **`ErrorCompat::iter_chain()`** — traverse an error and its sources recursively (call in fully-qualified form for future-proofing).

## Async integration

With the `futures` feature enabled, SNAFU provides `TryFutureExt` and `TryStreamExt` traits for attaching context in combinator chains without manual `map_err`:

```rust
fn example() -> impl TryFuture<Ok = i32, Error = Error> {
    another_function().context(AuthenticatingSnafu {
        user_name: "admin",
        user_id: 42,
    })
}
```

For [[tracing-error]] `SpanTrace` integration, define a local newtype implementing `GenerateImplicitData` and use `#[snafu(implicit)]` — see [[snafu-tutorial]] for the full pattern.

## Boundary conversions

SNAFU errors implement `std::error::Error + Send + Sync + 'static`, enabling clean boundary conversions:

- **To [[anyhow]]**: `?` converts SNAFU errors into `anyhow::Error` via `From` automatically.
- **To [[miette]]**: use `IntoDiagnostic` to convert into `miette::Report`. Note: may discard richer `Diagnostic` info if the error already implements it.
- **Pattern**: typed SNAFU errors inside modules, convert at the application boundary.

## Version and compatibility

- **v0.9.0** (2026-03-02): `Whatever` now `Send + Sync`, `snafu::Location` aliased to std `Location`
- **Edition 2018**, MSRV **1.65**, default compatibility target **Rust 1.81**
- `rust_1_81` feature uses `core::error::Error` instead of `std::error::Error`
- Default features: `["std", "rust_1_81"]`; `std` implies `alloc`
- `no_std + alloc` requires `rust_1_81` feature
- Dual-licensed MIT/Apache-2.0

## When to prefer Snafu over thiserror

Use Snafu when you need **enforced error granularity across a large workspace**. The context selector pattern makes it harder to take shortcuts — every `.context(SomeVariant)` call is visible and reviewable. For application-level errors, smaller projects, or maximum ecosystem familiarity, [[thiserror]] is simpler and universally known. For a detailed migration guide from thiserror to SNAFU, see [[snafu-tutorial]].

## Comparison with error-stack

Both Snafu and [[error-stack]] push toward structured error context, but differently. Snafu's structure lives in your error type definitions (context selectors per variant). error-stack keeps error types simple and builds structure in the report's frame stack at propagation time. Snafu is more natural for library APIs where the error type *is* the contract; error-stack is more natural for application-level error aggregation with arbitrary attachments.

See [[rust-error-crate-comparison]] for the full decision matrix.
