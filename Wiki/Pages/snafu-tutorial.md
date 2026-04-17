---
tags:
  - rust
  - error-handling
sources:
  - "Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md"
last_updated: 2026-04-16
---

# SNAFU Tutorial

A deep-dive tutorial on [[snafu]], covering its core mechanism (context selectors), progressive error design philosophy, performance patterns, async integration, and boundary conversions to [[anyhow]], [[miette]], and [[tracing-error]].

## Core mechanism: context selectors

`#[derive(Snafu)]` generates a **context selector** type for each error variant — a companion struct used to construct errors ergonomically via `ResultExt::context` and `with_context`. The selector strips `source` and `backtrace` fields (SNAFU handles those automatically) and applies `Into::into` on each field, so you can pass `&str` into a `String` field without explicit conversion.

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

This is SNAFU's signature move: the same `io::Error` source type mapped into two semantically distinct variants. Downstream logs immediately show *what operation* failed, not just that I/O failed.

## Selectors are private by default

Context selectors default to private visibility, reflecting SNAFU's opinion that modules should define cohesive error types locally. This reduces accidental coupling. If you need cross-module access, use `#[snafu(visibility(pub))]` on the error type or specific variants. Exporting selectors in a public API is warned against — their shape may change across SNAFU versions, creating semver hazards.

## Progressive error design

SNAFU explicitly recommends a graduation path:

1. **Start with `Whatever`** — turnkey string errors via `whatever!` and `ensure_whatever!` for maximum speed.
2. **Graduate to typed enums** — when you hit limitations or want structured context at call sites.
3. **Add opaque wrappers** — for public API stability, use a newtype wrapping an internal SNAFU enum with delegated traits and `From` conversion.

## Key macros

- `ensure!(condition, SelectorSnafu { fields })` — early return if condition fails
- `whatever!("message")` / `ensure_whatever!(condition, "message")` — turnkey string errors
- `.context(SelectorSnafu { fields })` — cheap context (fields already available)
- `.with_context(|err| SelectorSnafu { fields })` — lazy context (allocations/formatting only on error path; closure receives the source error for inspection)
- `OptionExt::context` — converts `Option<T>` into `Result<T, E>` with structured context

## Backtrace patterns

SNAFU's `Backtrace` type is env-gated: capture is a no-op unless `RUST_BACKTRACE` or `RUST_LIB_BACKTRACE` is set. The recommended patterns:

- **Leaf errors**: always include `backtrace: snafu::Backtrace` — capture cost is zero when disabled.
- **Hot-path errors**: use `backtrace: Option<snafu::Backtrace>` — captures only when the user explicitly enables it. Critical for errors used as flow control.
- **Chained errors**: don't add backtrace fields to errors with a `source` — the leaf's backtrace propagates through the chain.

## Reporting

`Display` prints only the top-level message. For full chain + backtrace output, use:

- `#[snafu::report]` on `main` — sugar over returning `Report`
- `snafu::Report` directly — works with `tokio::main`, `async_std::main`, etc.
- `CleanedErrorText` — lower-level tool for deduplicating chain messages

SNAFU's default `Display` no longer includes the source's `Display`, aligning with broader Rust error-handling guidance that chain messages should be composed by the reporter, not the error type.

## Async integration

With the `futures` feature enabled, SNAFU provides `TryFutureExt` and `TryStreamExt` for attaching context in combinator chains:

```rust
use snafu::prelude::*;

fn example() -> impl TryFuture<Ok = i32, Error = Error> {
    another_function().context(AuthenticatingSnafu {
        user_name: "admin",
        user_id: 42,
    })
}
```

For [[tracing-error]] `SpanTrace` integration, use a local newtype implementing `GenerateImplicitData` with `#[snafu(implicit)]`:

```rust
#[derive(Debug)]
struct MySpanTrace(SpanTrace);

impl snafu::GenerateImplicitData for MySpanTrace {
    fn generate() -> Self { MySpanTrace(SpanTrace::capture()) }
}

#[derive(Debug, Snafu)]
pub struct Error {
    message: String,
    #[snafu(implicit)]
    span: MySpanTrace,
}
```

## Boundary conversions

- **To [[anyhow]]**: SNAFU errors implement `std::error::Error + Send + Sync + 'static`, so `?` converts them into `anyhow::Error` automatically.
- **To [[miette]]**: use `IntoDiagnostic` to convert `Result<T, SnafuError>` into `Result<T, miette::Report>`. Note: if the error already implements `Diagnostic`, this conversion may discard richer info.
- **To either**: the pattern is always the same — typed SNAFU errors inside modules, convert at the application boundary.

## Migration from thiserror

The move from [[thiserror]] to SNAFU is less about better derives and more about adopting structured context at call sites:

1. Replace `#[derive(Error)]` + `#[error("...")]` with `#[derive(Snafu)]` + `#[snafu(display("..."))]`
2. Replace `map_err(|e| Error::Variant { source: e, ... })` with `.context(VariantSnafu { ... })`
3. Use `#[snafu(transparent)]` for transparent delegation (replaces thiserror's `#[error(transparent)]`)
4. For stable public APIs, adopt the opaque newtype pattern

## Version and compatibility

- **v0.9.0** (2026-03-02): `Whatever` now `Send + Sync`, `snafu::Location` aliased to std `Location`
- **Edition 2018**, MSRV **1.65**, tested back to 1.65, default compatibility target Rust 1.81
- `rust_1_81` feature uses `core::error::Error` instead of `std::error::Error`
- Dual-licensed MIT/Apache-2.0

See [[rust-error-crate-comparison]] for where SNAFU fits in the broader ecosystem.
