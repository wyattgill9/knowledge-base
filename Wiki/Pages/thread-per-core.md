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

# Thread-per-core Architecture

Thread-per-core (also called shard-per-core) is a server architecture pattern where each CPU core runs a single-threaded event loop with its own memory allocator, network queue, and storage partition. No work-stealing, no cross-thread synchronization. The pattern traces to ScyllaDB's [[seastar]] framework and solves a hardware problem: [[cache-coherency]] is the dominant cost in multi-threaded systems at scale.

## Why it works

When [[tokio]]'s work-stealing scheduler migrates a task from core A to core B, every cache line that task touches must be invalidated on A and reloaded on B via the [[cache-coherency|MESI protocol]] — **50–200 ns per cache line** under contention. False sharing (independent data on the same 64-byte line) can degrade throughput by 10x. Even uncontended atomic operations carry a 5x penalty over non-atomic equivalents due to LOCK prefix instructions. Thread-per-core eliminates all of this: ScyllaDB claims near-linear scaling because there is essentially no Amdahl's Law serial bottleneck from inter-core coordination.

## The queueing theory penalty

Josh Snyder's 2024 analysis quantifies the cost: at 960K RPS with 100 us average service time and a 50% latency tolerance, a shared queue (M/M/k) needs **98 CPUs**, while shard-per-core (independent M/M/1 queues) needs **288 CPUs** — a **3x overprovisioning penalty**. Each shard saturates independently with no mechanism to redistribute excess capacity. This is the fundamental trade-off: you gain throughput scaling but lose hardware efficiency.

## The "poor person's thread-per-core" with Tokio

Run N independent `tokio::runtime::Builder::new_current_thread()` instances with `SO_REUSEPORT`. This captures **1.5–2x improvement** over Tokio's multi-threaded default (validated by TechEmpower's top Rust entries) while retaining full ecosystem access — reqwest, axum, tonic, tower all work unchanged. This is the recommended approach for the majority of Rust async developers.

## In io_uring runtimes

[[monoio]], [[glommio]], and [[compio]] are designed around thread-per-core from the ground up. Every io_uring submission queue is per-thread, every buffer is thread-local, tasks are `Rc<Task>` not `Arc`. This is fundamental to their performance (~3x Tokio at 16 cores) but also to their constraints: you must design your application around data partitioning, and the entire [[tokio]] ecosystem is incompatible.

## Failure modes

- **Head-of-line blocking** — a compute-heavy task that fails to yield blocks all I/O on its core. Only [[glommio]] has stall detection; [[monoio]] and [[compio]] offer none.
- **Workload imbalance** — if data access is skewed (hot partitions, popular users), some cores saturate while others idle, with no redistribution mechanism. Monoio's docs explicitly acknowledge this.
- **RefCell footgun** — thread-per-core eliminates `Mutex` but tempts `RefCell`. Holding a borrow across `.await` allows the executor to yield to another future that borrows the same cell, causing a runtime panic. [[apache-iggy|Apache Iggy]] solved this with ECS-style Struct-of-Arrays decomposition.
