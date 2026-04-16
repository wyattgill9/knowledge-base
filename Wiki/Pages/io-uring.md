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

# io_uring

io_uring is Linux's completion-based async I/O interface. It submits I/O operations with pre-allocated buffers and receives completion notifications — versus [[tokio]]'s readiness model (epoll) which waits for readiness then performs the I/O. In theory, this reduces syscalls through batched submissions and enables true zero-copy with fixed kernel buffers.

## The cancellation safety crisis

A critical October 2024 analysis ("Async Rust is not safe with io_uring") demonstrated that **async Rust's cancellation model is fundamentally incompatible with io_uring**. All io_uring runtimes leak TCP connections when using `select!` for timeouts, because dropping a future doesn't cancel the kernel operation already in flight. [[monoio]]'s `Canceller` API mitigates this but is "contagious" — every future in the chain must handle cancellation explicitly, with no compile-time enforcement.

## Rust io_uring runtimes

- **[[monoio]]** (ByteDance, production-proven) — highest io_uring throughput, ~3x [[tokio]] at 16 cores.
- **[[glommio]]** (Datadog) — better ergonomics with three-ring architecture and share-based task scheduling.
- **[[compio]]** — the only cross-platform completion-based runtime (io_uring + IOCP + polling fallback).
- **tokio-uring** — effectively unmaintained, avoid.

## Recommendation

Use io_uring runtimes only for **Linux-exclusive, high-concurrency I/O workloads** (proxies, storage engines) where you accept [[thread-per-core]] architecture and manual cancellation handling. For everything else, Tokio in thread-per-core `current_thread` mode closes much of the gap without the hazards.
