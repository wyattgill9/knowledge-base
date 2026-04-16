---
tags:
  - rust
  - concurrency
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Monoio

Monoio is ByteDance's production-proven [[io-uring]]-native async runtime for Rust. It achieves the highest io_uring throughput among Rust runtimes — **~3x [[tokio]] at 16 cores** — using a [[thread-per-core]] architecture.

## Cancellation mitigation

Monoio's `Canceller` API addresses the [[io-uring#cancellation-safety|io_uring cancellation crisis]] but is "contagious" — every future in the chain must handle cancellation explicitly, with no compile-time enforcement. This is a significant ergonomic and correctness burden compared to Tokio's readiness-based model.

## When to use it

Only for **Linux-exclusive, high-concurrency I/O workloads** (proxies, storage engines) where you've benchmarked a clear advantage over Tokio in thread-per-core mode and accept manual cancellation handling. For cross-platform needs, see [[compio]].
