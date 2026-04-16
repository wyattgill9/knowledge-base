---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# iai-callgrind

iai-callgrind uses Valgrind's instruction-count measurements to provide **deterministic, single-shot benchmarks** that detect sub-1% regressions regardless of system load. This makes it ideal for CI environments where wall-clock benchmarks from [[divan]] or [[criterion]] would be noisy.

## How it works

Instead of measuring elapsed time (which varies with CPU frequency, load, and thermal throttling), iai-callgrind counts the exact number of instructions executed. This gives perfectly reproducible results — a 0.5% instruction count increase is a real regression, not noise.

## Trade-offs

iai-callgrind requires Valgrind (Linux/macOS only) and runs significantly slower than wall-clock benchmarks since it instruments every instruction. Use it in CI for regression gates, not for interactive development. Pair with [[divan]] for the full benchmarking workflow.
