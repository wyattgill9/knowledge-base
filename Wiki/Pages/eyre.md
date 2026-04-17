---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
last_updated: 2026-04-16
---

# eyre

eyre is a fork of [[anyhow]] for dynamic error handling in Rust, distinguished by its **swappable report handler** architecture. Where anyhow provides a fixed error report format, eyre lets you plug in custom handlers that control how errors are displayed, what context is captured, and how panics are presented.

## Handler ecosystem

eyre's handler system enables companion crates with different tradeoffs:

- **[[color-eyre]]** — colorful reports with SpanTrace and panic hooks. The most popular handler. Repository is archived but stable and widely used.
- **stable-eyre** — backtrace support on stable Rust.
- **simple-eyre** — minimal handler that captures no additional info beyond the error chain.

This modularity is eyre's key advantage over [[anyhow]]: you can swap presentation without changing error-handling code throughout your application.

## Public API warning

eyre's documentation explicitly warns against re-exporting `eyre` types in library public APIs. Because eyre introduces its own `Report` type (not just `std::error::Error`), re-exporting it creates version coupling and semver hazards. Libraries should use [[thiserror]] or [[snafu]] for typed error definitions; eyre belongs at the application boundary.

## When to use eyre vs anyhow

Use eyre when you want **customizable error presentation** — colors, sections, SpanTraces, or other handler-specific features. Use [[anyhow]] when you just need straightforward error propagation with string context and fewer dependencies.

For the full decision matrix, see [[rust-error-crate-comparison]] and [[rust-error-handling]].
