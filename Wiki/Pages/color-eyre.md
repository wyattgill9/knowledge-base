---
tags:
  - rust
  - error-handling
  - crate
sources:
  - "Raw/Rust/Comparative Deep Research on Rust Error-Handling Crates, Including Snafu.md"
last_updated: 2026-04-16
---

# color-eyre

color-eyre is an [[eyre]] report handler that produces colorful, multi-section error reports with integrated panic hooks. It is one of the most popular choices for CLI application error presentation, though its GitHub repository is now archived.

## Key features

color-eyre installs both an error report handler and a panic hook, producing consistent, visually rich output for both. It captures `SpanTrace` (via [[tracing-error]]) and `Backtrace`, with an important distinction: **SpanTrace capture is significantly cheaper than Backtrace capture** and is enabled by default.

A notable performance gotcha: [[eyre]] uses `std::backtrace::Backtrace` (precompiled with optimizations), while color-eyre depends on `backtrace::Backtrace`, whose debug build can be an order of magnitude slower to capture. The docs provide guidance for optimizing the `backtrace` crate in dev profiles.

## Maintenance status

The `eyre-rs` organization marks the color-eyre repository as archived. It is still widely used and functional, but watch for maintenance gaps if you're adopting it for a new project. The core functionality is stable enough that archival is not an immediate concern, but long-term alternatives like [[miette]] may be safer bets for new projects needing rich error presentation.

## Typical stack

color-eyre sits at the top of the error stack: library crates define typed errors with [[thiserror]] or [[snafu]], application code wraps them in [[eyre]]'s `Report`, and color-eyre handles the final rendering. See [[rust-error-crate-comparison]] for the full decision matrix.
