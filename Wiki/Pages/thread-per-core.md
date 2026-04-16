---
tags:
  - rust
  - performance
  - concurrency
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Thread-per-core Architecture

Thread-per-core is a server architecture pattern where each CPU core runs a single-threaded event loop with its own resources — no work-stealing, no cross-thread synchronization. This maximizes cache locality and eliminates lock contention at the cost of requiring explicit data partitioning.

## In Tokio

Run [[tokio]]'s `current_thread` runtime on each core instead of the default multi-threaded scheduler. TechEmpower benchmarks show **1.5–2x throughput** improvement. Each core handles its own connections and data, communicating via message passing only when necessary.

## In io_uring runtimes

[[monoio]] and [[glommio]] are designed around thread-per-core from the start. Every io_uring submission queue is per-thread, every buffer is thread-local. This is fundamental to their performance advantage (~3x Tokio at 16 cores) but also to their programming model constraints — you must design your application around data partitioning.

## Trade-offs

Thread-per-core is powerful but opinionated. It works well for stateless request handling (proxies, API servers) and partitionable data (sharded storage). It's awkward for workloads requiring frequent cross-core coordination or global shared state.
