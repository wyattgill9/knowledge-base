---
tags:
  - rust
  - concurrency
  - crate
  - performance
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Tokio

Tokio is Rust's dominant async runtime and the foundation of its networking ecosystem — [[axum]], [[tonic]], [[hyper]], and [[tower]] all build on it. As of 2026, no other runtime comes close to matching its ecosystem breadth, and it remains the correct default for async Rust.

## Performance tuning

The default multi-threaded work-stealing scheduler is convenient but leaves performance on the table. For maximum throughput, run Tokio's `current_thread` runtime in a [[thread-per-core]] pattern — TechEmpower benchmarks show **1.5–2x throughput** over the default scheduler due to better [[cache-coherency|cache locality]] and zero cross-thread synchronization.

The "poor person's thread-per-core" pattern: run N independent `tokio::runtime::Builder::new_current_thread()` instances with `SO_REUSEPORT`. This is validated by TechEmpower's top Rust entries and is **the recommended approach for most Rust developers** who want shard-per-core benefits without abandoning the Tokio ecosystem.

## Tokio vs io_uring runtimes

[[io-uring]]-native runtimes like [[monoio]] achieve ~3x Tokio throughput at 16 cores, but come with severe trade-offs:

- **Cancellation safety crisis** — dropping in-flight futures causes use-after-free in kernel space and TCP connection leaks.
- **Ecosystem incompatibility** — the ownership-transfer buffer model is fundamentally incompatible with Tokio's `AsyncRead`/`AsyncWrite` traits, meaning reqwest, axum, tonic, tower, and hundreds of other crates cannot be used.
- **Container blocker** — Docker/containerd block io_uring by default after security concerns.

Tokio's readiness-based model (epoll) avoids all of these. For file I/O, Tokio delegates to a blocking thread pool (up to 512 threads), which is 29x slower than io_uring for random reads — but adequate for most workloads.

## Ecosystem integration

The [[bytes]] crate is the standard buffer type throughout Tokio's ecosystem. [[tracing]] is the recommended diagnostics layer, and [[tokio-console]] provides real-time async task debugging. For message passing within Tokio, `tokio::sync::mpsc` benefits from intra-thread coroutine switching that can outperform standalone channels like [[kanal]] when sender and receiver share a worker thread.

## The ecosystem moat

Tokio's deepest advantage isn't performance — it's that the entire Rust async ecosystem builds on its traits and runtime. Migrating to [[compio]], [[monoio]], or [[glommio]] means abandoning this ecosystem or writing compatibility shims that negate performance gains. This lock-in is also a feature: any library you pick up just works.
