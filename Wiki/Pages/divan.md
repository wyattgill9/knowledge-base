---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# divan

divan is the recommended benchmarking crate for new Rust projects. Its `#[divan::bench]` attribute is as simple as `#[test]`, with built-in parameterization, allocation counting, and module-based tree output.

## Why divan over criterion

[[criterion]] remains the mature statistical option with HTML reports, but has seen reduced maintainer activity. divan offers a more ergonomic API, better default output, and active development. For CI environments where wall-clock benchmarks are noisy, complement divan with [[iai-callgrind]] for deterministic instruction-count measurements.

## The benchmarking stack

A complete Rust benchmarking setup uses **divan** for development iteration (fast, readable output), **[[iai-callgrind]]** for CI regression detection (deterministic, sub-1% sensitivity), and **[[cargo-flamegraph]]** + **[[samply]]** for profiling when benchmarks reveal a problem. Add **[[dhat-rs]]** for heap profiling to track allocations, peak memory, and hot allocation sites.
