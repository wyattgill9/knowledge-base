---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Snafu

Snafu is a Rust [[rust-error-handling|error-handling]] crate that uses context selectors to enforce granular, per-module error types with structured backtraces. It remains a strong choice for large multi-crate workspaces — GreptimeDB, InfluxDB IOx, and Iroh all use it — but [[thiserror]] 2.0 has overtaken it as the ecosystem default.

## Context selectors

Snafu's distinguishing feature is context selectors: each error variant gets a companion type that attaches context at the call site. This is more verbose than thiserror (~5 lines per variant vs ~2) but forces developers to create specific error types per failure mode rather than lumping errors into catch-all variants. In large codebases, this discipline pays off with more informative error chains.

## When to prefer Snafu over thiserror

Use Snafu when you need **enforced error granularity across a large workspace**. The context selector pattern makes it harder for contributors to take shortcuts with error types — every `.context(SomeVariant)` call is visible and reviewable. For application-level errors or smaller projects, [[thiserror]] 2.0 is simpler, less verbose, and universally familiar.

Neither crate has meaningful runtime overhead — the choice is purely about API design and team discipline. For dynamic error types (prototyping, scripts), pair either with [[anyhow]] (stable) or [[eyre]] (richer reports).
