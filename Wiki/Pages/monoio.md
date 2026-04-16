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

# Monoio

Monoio is ByteDance's [[thread-per-core]] async runtime for Rust, open-sourced through CloudWeGo and deployed in production powering their Monolake proxy framework. Its defining architectural decision is the **ownership-transfer buffer model**, which enables true zero-copy I/O at the cost of [[tokio]] ecosystem compatibility.

## Architecture

Each CPU core runs an independent `Runtime<D>` with its own event loop, task queue (`VecDeque`), and [[io-uring]] ring. Tasks are `Rc<Task>` (not `Arc`), spawned futures need not be `Send` or `Sync`, and multi-core deployment uses `SO_REUSEPORT` so each thread's listener accepts connections independently — the kernel distributes incoming connections, eliminating cross-thread work distribution entirely.

The `AsyncReadRent` and `AsyncWriteRent` traits are Monoio's most consequential design choice. Where Tokio's `AsyncRead::poll_read` borrows a `&mut [u8]`, Monoio's `read<T: IoBufMut>(buf: T) -> BufResult<usize, T>` **takes ownership** of the buffer and returns it alongside the result. This is necessary because io_uring requires stable buffer addresses — the kernel holds a pointer during the async operation, and allowing the buffer to move or drop would cause use-after-free in kernel space. The cost: incompatibility with Tokio's entire ecosystem (reqwest, axum, tonic, tower).

## I/O allocation strategy

Monoio tracks in-flight operations via a `Slab<Lifecycle>` indexed by slot number, mapping directly to io_uring's `user_data` field. This slab-based allocation avoids heap allocation per I/O operation — a concrete advantage over [[compio]]'s boxing approach. The `FusionDriver` probes for io_uring at startup and falls back to mio-based epoll/kqueue transparently.

## Scheduler

Purely cooperative with **no fairness guarantees**. The main loop drains the task queue in FIFO order with no budget system analogous to Tokio's `coop` module — a compute-heavy task that never yields will starve all other tasks on that core. No stall detection (unlike [[glommio]]). `yield_now()` is the only escape hatch, and discipline falls entirely on the developer.

## Performance

At **4 cores, ~2x Tokio; at 16 cores, ~3x Tokio**. Throughput keeps scaling where Tokio plateaus due to work-stealing contention. File I/O benchmarks (Tonbo/Fusio) show 4KB random reads in **3.78 us vs Tokio's 110 us** — a 29x improvement from true async I/O vs Tokio's blocking thread pool. However, at very low connection counts on a single core, Monoio's latency is *higher* than Tokio's because io_uring's ring setup and batched submission adds overhead that epoll's direct readiness check avoids.

ByteDance's Monolake gateway outperformed NGINX by 20%, and their RPC layer showed 26% improvement over Tokio-based equivalents.

## Cancellation safety

Monoio's `Canceller` API addresses the [[io-uring#cancellation-safety|io_uring cancellation crisis]] but is "contagious" — every future in the chain must handle cancellation explicitly, with no compile-time enforcement.

## When to use it

Choose Monoio over [[compio]] only if you are building a **Linux-only network proxy** where the slab allocator's zero-allocation I/O submission provides measurable benefit in your specific hot path, and you are comfortable with ByteDance's slower maintenance cadence. For most new io_uring projects, Compio is the recommended default. For most Rust async projects, [[tokio]] in thread-per-core mode is the pragmatic choice.
