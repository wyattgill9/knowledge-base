---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# kanal

kanal is a high-throughput message passing channel for Rust, leading benchmarks at **8–16M msg/sec** for bounded async operations. It provides a unified sync/async API, avoiding the need for separate channel types.

## The channel landscape

- **kanal** — fastest throughput, unified sync/async API.
- **`tokio::sync::mpsc`** — benefits from intra-thread coroutine switching within [[tokio]], potentially outperforming kanal when sender and receiver share a worker thread.
- **[[crossbeam-channel]]** — the proven choice for synchronous MPMC (multiple producer, multiple consumer) scenarios.

Choose kanal for maximum raw throughput. Choose `tokio::sync::mpsc` when already inside Tokio and colocation is likely. Choose crossbeam-channel for sync-only MPMC.
