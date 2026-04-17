---
tags:
  - rust
  - error-handling
  - architecture
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
  - "Raw/Rust/SNAFU Deep Research Tutorial - Structured, Context-Rich Errors in Rust.md"
last_updated: 2026-04-16
---

# Rust Error Crate Comparison

A comprehensive comparative analysis of the Rust error-handling crate ecosystem as of early 2026, covering typed error definition, type-erased reporting, diagnostic rendering, and observability integration.

## The two strategic divides

Two decisions dominate every Rust project's error strategy:

**Library vs application boundary.** Libraries should expose typed, stable error types that don't leak implementation details. Applications should convert to type-erased reports at their boundary and focus on human-readable context. [[thiserror]] and [[snafu]] serve the library side; [[anyhow]] and [[eyre]] serve the application side. Mixing these roles — e.g., returning `anyhow::Error` from a library's public API — is an anti-pattern that [[eyre]]'s own docs explicitly warn against.

**String context vs structured context.** [[anyhow]] and [[eyre]] attach context as strings — fast to write, cheap to propagate, but opaque to programmatic inspection. [[snafu]] and [[error-stack]] push structured, semantically named context that acts as a "semantic backtrace" of what the program was attempting at each layer. The structured approach costs more upfront modeling but scales better in large systems where you need observability, not just error messages. SNAFU's context selectors enforce this structurally: each variant gets a generated companion type with `Into::into` field conversions, private by default to encourage module-local error design. See [[snafu-tutorial]] for the full tutorial with progressive examples and boundary conversion patterns.

## The four ecosystem layers

The ecosystem has converged into four complementary layers rather than a single "best crate":

1. **Typed error definition** — [[thiserror]], [[snafu]], [[derive-more]] `Error`, [[displaydoc]]
2. **Type-erased reporting** — [[anyhow]], [[eyre]], [[color-eyre]]
3. **Diagnostic rendering** — [[miette]], [[ariadne]], [[codespan-reporting]]
4. **Observability context** — [[tracing-error]], [[color-eyre]]'s `SpanTrace` integration

Most production stacks combine crates across these layers. A typical binary uses [[anyhow]] or [[eyre]] at the top level with [[thiserror]] or [[snafu]] for internal modules. Observability-first services add [[tracing-error]] for async span context. Compiler-like tools add [[miette]] or [[ariadne]] for source-spanned diagnostics.

## Decision matrix

| Use case | Primary choice | Why |
|---|---|---|
| CLI apps (nice output) | [[anyhow]] + [[color-eyre]] or [[miette]] | anyhow for easy propagation; color-eyre/miette for presentation |
| Libraries (public API stability) | [[thiserror]] or [[snafu]] | standard trait impls, no type erasure in public API |
| Production services (observability) | [[snafu]] or [[error-stack]] + [[tracing-error]] | structured context, SpanTrace beats backtraces in async |
| `no_std` / embedded | [[snafu]] (feature-tuned), [[derive-more]], hand-rolled enums | explicit alloc/std feature splits |
| Prototyping / internal tooling | [[anyhow]] or [[eyre]] | maximum development speed |
| Parser/compiler UX (source spans) | [[miette]] or [[ariadne]] / [[codespan-reporting]] | built for "show the user where and why" |

## Recommended organizational pattern

A simple, scalable policy for teams:

- **Library crates**: define domain errors with [[thiserror]] (minimum ceremony) or [[snafu]] (structured context selectors). Keep error types stable and intentional.
- **Binary/application crates**: convert to `anyhow::Result` or `eyre::Result` at the boundary. Add human-readable context. Use [[miette]] if you need compiler-grade UX.
- **Observability-first services**: use [[tracing-error]] + SpanTraces for async context. Consider [[error-stack]] attachments for carrying structured payloads through the error path.

## Legacy crates to avoid

[[failure]] is deprecated (RustSec EOL). [[error-chain]] is unmaintained. `fehler` was an experiment that is no longer maintained. `err-derive` is superseded by [[thiserror]] and [[derive-more]]. `quick-error` still works but modern derive-driven solutions are universally preferred. See [[rust-error-handling]] for the full landscape.
