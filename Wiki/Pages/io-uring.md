---
tags:
  - rust
  - performance
  - concurrency
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# io_uring

io_uring is Linux's completion-based async I/O interface. Userspace submits I/O requests to a Submission Queue and harvests results from a Completion Queue — both lock-free ring buffers in shared memory. **Multiple operations batch into a single `io_uring_enter` syscall**, and checking for completions requires only comparing two integers (head and tail pointers), with no syscall at all. This contrasts with [[tokio]]'s readiness model (epoll), which notifies that a socket is readable, then userspace issues a separate `read()` syscall.

For file I/O, the difference is starker: Tokio considers files "always ready" under epoll and delegates to a blocking thread pool (up to 512 threads), while io_uring performs truly asynchronous file operations — yielding **29x improvements** in random read benchmarks (3.78 us vs 110 us for 4KB reads).

## The cancellation safety crisis

A critical October 2024 analysis ("Async Rust is not safe with io_uring") demonstrated that **async Rust's cancellation model is fundamentally incompatible with io_uring**. The kernel holds a pointer to the user's buffer during the async operation. Dropping a future without cancelling the io_uring operation causes use-after-free in kernel space. All io_uring runtimes leak TCP connections when using `select!` for timeouts. [[monoio]]'s `Canceller` API mitigates this but is "contagious" — every future in the chain must handle cancellation explicitly, with no compile-time enforcement.

## Buffer ownership model

All Rust io_uring runtimes use ownership-transfer APIs: `read<T: IoBufMut>(buf: T) -> BufResult<usize, T>` takes ownership of the buffer and returns it with the result. This guarantees pointer stability for the kernel but creates **fundamental incompatibility with [[tokio]]'s ecosystem** — reqwest, axum, tonic, tower, and hundreds of other crates cannot be used without compatibility shims that negate performance gains.

## Container security blocker

Docker, containerd, and other major container runtimes **block io_uring by default** in their seccomp profiles. This follows ARMO researchers' demonstration of a rootkit operating entirely through io_uring syscalls that bypassed all major runtime security tools. Any containerized deployment requires custom seccomp profiles — an operational burden that may be unacceptable in enterprise environments.

## Rust io_uring runtimes

- **[[monoio]]** (ByteDance) — highest network throughput via slab-allocated zero-copy I/O. Best for Linux-only proxies.
- **[[compio]]** (community) — **recommended default in 2026**. Cross-platform, pluggable driver-executor, actively maintained, validated by [[apache-iggy|Apache Iggy]] at 5M msg/sec.
- **[[glommio]]** (DataDog) — unmatched storage I/O with triple-ring architecture and proportional-share scheduling. Effectively unmaintained.
- **tokio-uring** — effectively unmaintained, avoid.

## Decision framework

1. **Does your workload naturally shard?** If not, work-stealing outperforms shard-per-core due to [[thread-per-core#queueing-theory|queueing theory]] (3x overprovisioning penalty).
2. **Are you on Linux with io_uring support and not in locked-down containers?** If not, only Compio offers a path, with reduced benefits.
3. **Can you afford to abandon the Tokio ecosystem?** If your service depends on Tokio-native libraries, the migration cost likely exceeds the performance gain.

For teams that don't pass all three gates, Tokio in [[thread-per-core]] `current_thread` mode captures 1.5–2x improvement with full ecosystem access.
