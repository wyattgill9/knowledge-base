---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# tikv-jemallocator

tikv-jemallocator is the maintained Rust wrapper for jemalloc, providing the most stable allocation latency across all size classes and thread counts. It includes built-in heap profiling via `tikv-jemalloc-ctl`.

## When to choose jemalloc

jemalloc is the better choice for **servers with mixed allocation sizes** — its arena-based design handles the full spectrum from tiny allocations to large mmap'd regions with consistent latency. The built-in profiling (`tikv-jemalloc-ctl`) lets you inspect heap state without external tools.

For cross-platform builds or workloads dominated by small allocations (<= 4 KB), [[mimalloc]] may be faster. On musl targets, either is nearly mandatory. See [[rust-memory-allocators]] for the full picture.
