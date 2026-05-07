---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# crossbeam-channel

crossbeam-channel is the proven choice for synchronous MPMC (multiple producer, multiple consumer) message passing in Rust. For async channels or maximum throughput, see [[kanal]]. For within-[[tokio]] communication, `tokio::sync::mpsc` may outperform standalone channels due to intra-thread coroutine switching. See [[rust-concurrent-data-structures]].

## Channel vs raw queue

A channel is a queue plus blocking semantics, notification, and `Send + Sync` safety. When you need just the queue, drop to [[crossbeam-array-queue|crossbeam `ArrayQueue`]] (bounded MPMC) or [[rtrb]] (wait-free SPSC). The throughput difference is significant: bare bounded queues run 5–50× faster than channels for the same topology because they skip the blocking and notification overhead. See [[the-fastest-queue]] and [[concurrent-queues]] for context.
