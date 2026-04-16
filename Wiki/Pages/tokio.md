---
tags:
  - rust
  - concurrency
  - crate
  - performance
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Tokio

Tokio is Rust's dominant async runtime and the foundation of its networking ecosystem — [[axum]], [[tonic]], [[hyper]], and [[tower]] all build on it. As of 2026, no other runtime comes close to matching its ecosystem breadth, and it remains the correct default for async Rust.

## Performance tuning

The default multi-threaded work-stealing scheduler is convenient but leaves performance on the table. For maximum throughput, run Tokio's `current_thread` runtime in a [[thread-per-core]] pattern — TechEmpower benchmarks show **1.5–2x throughput** over the default scheduler due to better cache locality and zero cross-thread synchronization.

## Tokio vs io_uring runtimes

[[io-uring]]-native runtimes like [[monoio]] achieve ~3x Tokio throughput at 16 cores, but face a fundamental cancellation safety crisis: dropping in-flight futures doesn't cancel the kernel operation, causing TCP connection leaks with `select!`-based timeouts. Tokio's readiness-based model (epoll) avoids this entirely. Use Tokio unless you've benchmarked io_uring on your specific workload and accept the trade-offs.

## Ecosystem integration

The [[bytes]] crate is the standard buffer type throughout Tokio's ecosystem. [[tracing]] is the recommended diagnostics layer, and [[tokio-console]] provides real-time async task debugging. For message passing within Tokio, `tokio::sync::mpsc` benefits from intra-thread coroutine switching that can outperform standalone channels like [[kanal]] when sender and receiver share a worker thread.
