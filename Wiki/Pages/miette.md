---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md"
last_updated: 2026-04-16
---

# miette

miette is a diagnostic-first Rust [[rust-error-handling|error-handling]] crate that extends `std::error::Error` with rich metadata — source spans, labels, help text, error codes — and renders them as pretty, compiler-style diagnostics. Think of it as the presentation layer for errors that need to show the user *where* and *why* something failed.

## Core design

miette's `Diagnostic` trait (and its derive macro) lets you attach structured metadata to any error type:

```rust
use miette::{Diagnostic, NamedSource, SourceSpan};
use thiserror::Error;

#[derive(Debug, Error, Diagnostic)]
#[error("parse error")]
pub struct ParseError {
    #[source_code]
    src: NamedSource,
    #[label("here")]
    span: SourceSpan,
    #[help]
    help: Option<String>,
}
```

This pairs naturally with [[thiserror]] — you define your error type with thiserror's `#[derive(Error)]` and add miette's `#[derive(Diagnostic)]` for the metadata layer. miette also provides its own `Result`/`Report` types and a `miette!` macro analogous to [[anyhow]]'s `anyhow!`.

## The "fancy" feature

miette's rich terminal output (colors, unicode box-drawing, source underlining) lives behind the `"fancy"` feature flag, which pulls in additional dependencies. The docs explicitly recommend enabling `"fancy"` **only in the top-level crate** — libraries should derive `Diagnostic` without enabling fancy rendering, letting the application choose its presentation.

## When to use miette

miette is the right choice for **parsers, compilers, linters, config validators, and any tool where errors reference specific locations in source text**. For general application error handling without source spans, [[anyhow]] or [[eyre]] are simpler. For library error types, [[thiserror]] or [[snafu]] are more appropriate as the foundation, with miette layered on top when needed.

## Integration with SNAFU

[[snafu]] errors implement `std::error::Error + Send + Sync + 'static`, so miette's `IntoDiagnostic` trait provides a clean boundary adapter:

```rust
use miette::{IntoDiagnostic, Result};

fn main() -> Result<()> {
    mylib::run().into_diagnostic()?; // SnafuError -> miette::Report
    Ok(())
}
```

Be aware that converting an error that already implements `Diagnostic` via `IntoDiagnostic` may discard richer diagnostic info. For plain SNAFU error types without `Diagnostic` metadata, this is the cleanest boundary pattern. See [[snafu-tutorial]] for the full integration story.

## Alternatives

[[ariadne]] and [[codespan-reporting]] serve the same diagnostic rendering niche. ariadne emphasizes pretty output with fine-grained label control. codespan-reporting provides a more structured `Diagnostic` + labels + emit pipeline. All three are viable; miette is the most integrated with the broader error trait ecosystem.
