---
tags:
  - rust
  - error-handling
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/Modern Rust - the definitive 2023–2026 feature and idiom guide.md"
last_updated: 2026-04-16
---

# Rust Error Handling

The Rust error-handling ecosystem has converged into four complementary layers rather than a single "best crate." A key enabler: **`Error` moved to `core`** in Rust 1.81 (September 2024), enabling `no_std` error types without external crates — see [[modern-rust-features]]. The fundamental organizing principle is a two-layer architecture: typed errors for libraries, type-erased reports for applications. Within each layer, the key strategic choice is string context (fast, flexible, opaque) vs structured context (more modeling upfront, better debugging at scale). See [[rust-error-crate-comparison]] for the detailed comparative analysis.

## Layer 1: Typed error definition (libraries)

- **[[thiserror]] 2.0** — the ecosystem standard. Generates the same `Error` and `Display` impls you'd write by hand. Deliberately invisible in public API — switching between handwritten impls and thiserror is not a breaking change. `no_std` since 2.0. ~2 lines per variant. Use this by default.
- **[[snafu]]** — context selectors enforce per-module error granularity. Each variant gets a generated companion type for structured context attachment. More verbose (~5 lines per variant) but prevents error type degradation in large workspaces. The context selectors act as a "semantic backtrace" — what the program was attempting at each layer. Used by GreptimeDB, InfluxDB IOx, Iroh.
- **[[derive-more]] `Error`** — multi-trait derive crate with `Error` support. Viable when you want one derive crate for Display, From, Error, etc. `no_std` compatible.
- **[[displaydoc]]** — derives `Display` from doc comments. Pure formatting helper, `no_std`.

Neither thiserror nor snafu has meaningful runtime overhead. The choice is purely API design philosophy: familiarity and conciseness (thiserror) vs enforced discipline and structured context (snafu).

## Layer 2: Type-erased reporting (applications)

- **[[anyhow]]** — stable, simple `anyhow::Result` boxing. The default for application code. Works with any `std::error::Error`. Provides `Context`, `with_context`, `bail!`, `anyhow!`. Supports `no_std` (with allocator). Environment-variable gated backtraces on stable.
- **[[eyre]]** — fork of anyhow with swappable report handlers. Better when you want customizable error presentation. Explicitly warns against re-exporting in library public APIs due to semver hazards.
- **[[color-eyre]]** — eyre report handler producing colorful reports with SpanTrace and panic hooks. Repository is archived but widely used. SpanTrace capture is cheaper than Backtrace capture and enabled by default.

## Layer 3: Diagnostic rendering (parsers, compilers, CLI UX)

- **[[miette]]** — diagnostic-first errors with source spans, labels, help text, error codes. Enable the `"fancy"` feature only in top-level crates. Can replace anyhow/eyre-style `Result`/`Report`.
- **[[ariadne]]** — pretty diagnostic rendering with fine-grained label control.
- **[[codespan-reporting]]** — structured diagnostic pipeline. Actively maintained (0.13.1, October 2025).

## Layer 4: Observability context (async services)

- **[[tracing-error]]** — enriches errors with [[tracing]] span context. `SpanTrace` shows the logical application hierarchy instead of executor frames, making it far more useful than backtraces in async code. Marked "experimental."
- **[[error-stack]]** — stacked `Report` with typed contexts and arbitrary attachments. More than a source chain — you can carry opaque structured payloads. Deliberately adds development overhead for debugging payoff. Maintained by HASH.

## Legacy crates to avoid

- **[[failure]]** — deprecated, RustSec EOL. Migrate to [[anyhow]]/[[thiserror]].
- **[[error-chain]]** — unmaintained since 2020. Migrate to [[thiserror]]/[[snafu]] + [[anyhow]]/[[eyre]].
- **fehler** — experimental, no longer maintained.
- **err-derive** — superseded by [[thiserror]] and [[derive-more]].
- **quick-error** — functional but last release 2021; modern derive solutions preferred.

## Recommendation

For most projects: [[thiserror]] at library boundaries, [[anyhow]] for application plumbing. Switch to [[snafu]] in large workspaces where error discipline matters. Add [[miette]] for compiler-style UX, [[tracing-error]] for async observability, or [[error-stack]] when you need structured attachments on the error path.
