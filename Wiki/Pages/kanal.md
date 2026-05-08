---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# kanal

kanal is a high-throughput message passing channel for Rust, leading the channel benchmarks at **8–16M msg/sec** for bounded async operations. It provides a unified sync/async API, avoiding the need for separate channel types.

## The channel landscape

- **kanal** — fastest throughput, unified sync/async API.
- **`tokio::sync::mpsc`** — benefits from intra-thread coroutine switching within [[tokio]], potentially outperforming kanal when sender and receiver share a worker thread.
- **[[crossbeam-channel]]** — the proven choice for synchronous MPMC (multiple producer, multiple consumer) scenarios.

Choose kanal for maximum raw throughput. Choose `tokio::sync::mpsc` when already inside Tokio and colocation is likely. Choose crossbeam-channel for sync-only MPMC.

## Throughput in global context

8–16M msg/sec is fast for a Rust *channel* (which carries messaging semantics, not just queueing), but lower than what bare [[mpmc-queue|MPMC queues]] achieve. The C++ [[atomic-queue|`atomic_queue`]] hits 200–500M+ ops/s in 1P/1C, and an optimized [[spsc-queue|SPSC]] [[ring-buffer]] reaches 520M+ ops/s on Apple M4. The gap exists because channels add: blocking semantics with notification, MPMC coordination, and message-typing safety. When raw queue throughput matters more than channel ergonomics, drop down to [[crossbeam-array-queue|crossbeam ArrayQueue]] for MPMC or [[rtrb]] for SPSC. See [[the-fastest-queue]] for the full hierarchy.

## Channels vs Go channels

A common comparison: Go's built-in goroutine channels are convenient but significantly slower than dedicated lock-free queues. They're optimized for ease of use and ecosystem integration, not raw throughput. Rust's kanal is roughly 5–10× faster than Go's `chan` at equivalent semantics, primarily because it avoids the goroutine scheduler interaction Go channels require. This is the typical pattern at the [[inter-thread-communication|inter-thread layer]]: language-level convenience features add fixed costs that bare lock-free queues avoid. See [[mechanical-sympathy]] for why this gap is structural rather than incidental.
