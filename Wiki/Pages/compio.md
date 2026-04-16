---
tags:
  - rust
  - concurrency
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Compio

Compio, authored by Yuyi Wang (Berrysoft), is the **recommended [[thread-per-core]] runtime for new [[io-uring]] projects in 2026**. It is the only shard-per-core Rust runtime with first-class support for both io_uring and Windows IOCP, plus a polling fallback for macOS. Its architecture disaggregates the I/O driver from the executor — a design choice that proved decisive for its most significant adopter, [[apache-iggy|Apache Iggy]].

## Pluggable driver-executor architecture

The `compio-driver` crate provides a platform-agnostic `Proactor` with `push()`, `poll()`, `pop()`, and `cancel()`. On Linux it wraps io_uring with what multiple evaluations describe as the **broadest io_uring feature coverage** among Rust runtimes — buffer pools, personalities, splice operations, statx, and active tracking of new kernel features. On Windows, it wraps IOCP natively. The Proactor is `!Send` and `!Sync`, enforcing single-threaded-per-core semantics at the type level.

The driver-executor separation is Compio's key differentiator. Users can build custom executors while reusing Compio's I/O driver. This enables [[deterministic-simulation-testing]] (DST) — replacing the time wheel, scheduler, and driver with deterministic mocks for reproducible testing of distributed system behaviors. Neither [[monoio]] nor [[glommio]] supports this.

## I/O allocation trade-off

Compio **boxes each I/O request** submitted to the queue, incurring a heap allocation per operation. [[monoio]]'s slab allocator avoids this entirely. The trade-off was deliberate: slab allocation would add complexity to the cross-platform executor, particularly for the IOCP backend. [[apache-iggy|Apache Iggy]]'s team found the overhead negligible when using [[mimalloc]], confirming that for small, predictable allocations the cost is dominated by I/O latency.

## Buffer ownership model

Like Monoio, Compio uses ownership transfer: `read<B: IoBufMut>(buf: B) -> BufResult<usize, B>`. This creates the same [[tokio]] ecosystem incompatibility — when Iggy needed WebSocket support, they discovered `async-tungstenite` was fundamentally coupled to poll-based I/O and had to create `compio-ws` from scratch.

## Apache Iggy validation

[[apache-iggy|Apache Iggy]]'s migration from Tokio to Compio (completed December 2025) is the most thoroughly documented production adoption of any shard-per-core runtime. Results: **5,000 MB/s throughput** (5M messages/sec at 1KB), sub-millisecond P99 latency, ~9.5ms P9999 WebSocket latency with fsync-per-message persistence. They chose Compio over Monoio (slower maintenance, io_uring feature gaps) and Glommio (effective abandonment, design disagreements), reporting patches merged "within hours."

## When to use it

Compio is the right choice when you need completion-based I/O and any of: cross-platform support, architectural flexibility for testing, or active maintenance. It's a poor fit for workloads requiring [[glommio]]-level scheduler sophistication or where per-operation heap allocation is unacceptable in extreme hot paths.
