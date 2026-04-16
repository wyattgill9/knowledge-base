---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# crossbeam-channel

crossbeam-channel is the proven choice for synchronous MPMC (multiple producer, multiple consumer) message passing in Rust. For async channels or maximum throughput, see [[kanal]]. For within-[[tokio]] communication, `tokio::sync::mpsc` may outperform standalone channels due to intra-thread coroutine switching. See [[rust-concurrent-data-structures]].
