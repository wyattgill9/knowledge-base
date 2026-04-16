---
tags:
  - rust
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Shard-per-core Runtimes Compared

A detailed comparison of the three Rust async runtimes — [[monoio]], [[compio]], and [[glommio]] — that reject [[tokio]]'s work-stealing model in favor of [[thread-per-core]] execution. All three achieve **2–3x Tokio's throughput at 16+ cores** by eliminating [[cache-coherency]] traffic and atomic contention, but pay for this with queueing-theory penalties, ecosystem fragmentation, and cooperative scheduling risks.

## Comparative architecture

| Dimension | [[monoio|Monoio]] | [[compio|Compio]] | [[glommio|Glommio]] |
|---|---|---|---|
| **Creator** | ByteDance | Community (Berrysoft) | DataDog (Glauber Costa) |
| **io_uring rings/thread** | 1 | 1 | 3 (main/latency/poll) |
| **I/O allocation** | Slab (zero heap alloc) | Box per op | Internal managed buffers |
| **Buffer model** | Ownership transfer (zero-copy) | Ownership transfer | Copy into internal buffers |
| **Scheduler** | FIFO, no fairness | FIFO, no fairness | Proportional-share with latency classes |
| **Platform support** | Linux, macOS, Windows (experimental) | **Linux, Windows (IOCP), macOS** | Linux only (kernel >= 5.8) |
| **Driver-executor** | Coupled | **Decoupled** (pluggable) | Coupled |
| **Storage I/O** | Basic async file ops | Basic async file ops | DMA files, [[direct-io]], read-ahead |
| **Stall detection** | None | None | Built-in with stack traces |
| **Maintenance (2026)** | Active but slowing | **Very active** | Effectively unmaintained |
| **Production users** | ByteDance (Monolake) | [[apache-iggy|Apache Iggy]] | DataDog (internal) |

## Recommendation

**[[compio]] is the recommended default for new [[io-uring]] projects in 2026.** It's the only actively maintained option, the only one supporting Windows and macOS, and the only one with driver-executor separation enabling [[deterministic-simulation-testing]]. Choose [[monoio]] only for Linux-only network proxies where slab allocation matters. Choose [[glommio]] only for storage engines needing its unique scheduler and DMA abstractions — and prepare to fork it.

For the majority of Rust developers, the "poor person's thread-per-core" pattern — N independent `tokio::runtime::Builder::new_current_thread()` instances with `SO_REUSEPORT` — delivers 1.5–2x improvement with full [[tokio]] ecosystem access.
