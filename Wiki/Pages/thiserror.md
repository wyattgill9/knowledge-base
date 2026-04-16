---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# thiserror

thiserror is the Rust ecosystem's standard derive macro for defining custom error types. Version 2.0, released late 2024, added `no_std` support, DST handling, and improved trait bound inference — closing the feature gap with [[snafu]] and cementing its position as the default choice.

## Why thiserror won

The API is minimal: derive `Error`, annotate variants with `#[error("...")]` and `#[from]`/`#[source]`, and you're done — roughly **2 lines per variant vs 5 for Snafu**. This conciseness, combined with adoption by cargo, ruff, uv, tauri, and virtually every major Rust project, means any Rust developer can read and contribute to thiserror-based error types immediately.

## When Snafu is still better

[[snafu]]'s context selectors enforce more granular error types by design. In large multi-crate workspaces, this forced discipline prevents error type degradation over time. For smaller projects or application-level errors, thiserror's simplicity wins.

## Companion crates

For dynamic error types (prototyping, CLIs, scripts), pair thiserror with [[anyhow]] for stable, simple error boxing, or [[eyre]] for richer error reports with custom handlers. See [[rust-error-handling]] for the full landscape.
