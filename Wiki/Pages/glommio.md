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

# Glommio

Glommio is the most architecturally ambitious [[thread-per-core]] Rust runtime, designed by Glauber Costa at DataDog after seven years building ScyllaDB's [[seastar]] framework. It features a triple [[io-uring]] ring design, proportional-share scheduling with latency classes, and [[direct-io]] as a first-class citizen — capabilities no other Rust runtime offers. It is also **effectively unmaintained** since Costa's departure to co-found Turso in 2021.

## Triple-ring architecture

Each thread operates three independent io_uring instances:

- **Main Ring** — throughput-oriented operations.
- **Latency Ring** — time-sensitive tasks; a timer ensures yield within a specified duration.
- **Poll Ring** — NVMe operations using io_uring's interrupt-free polling mode.

Before the main ring blocks, the latency ring's file descriptor is registered for poll on the main ring, ensuring latency-sensitive completions wake sleeping threads. Checking the latency ring costs only two integer comparisons (CQ head and tail pointers), making `yield_if_needed()` nearly free.

## Proportional-share scheduling

Task queues have configurable `Shares`: if queue A has 2 shares and queue B has 1, A receives 2/3 of CPU time under contention. Shares can be static or dynamic, adjusted at runtime by **embedded controllers** that observe metrics and auto-tune weights — a feedback loop for self-tuning storage systems. Each queue declares a `Latency` class: `NotImportant` (throughput-optimized) or `Matters(Duration)` (latency-sensitive, with io_uring timer-driven preemption). Default preemption timeslice is 100ms. A stall detection handler sends `SIGUSR1` and collects stack traces when tasks exceed their expected runtime — observability neither [[monoio]] nor [[compio]] provides.

## Storage I/O

Glommio's storage subsystem is unmatched. `DmaFile` provides random-access [[direct-io]] that bypasses the OS page cache entirely, allocating from io_uring's pre-registered buffer pool and returning reference-counted pointers for zero-copy device-to-user transfer. `DmaStreamReader` implements intelligent read-ahead that dispatches new reads as each buffer is consumed, maintaining constant I/O parallelism.

Costa's benchmarks on Intel Optane: sequential scans of a 53GB file (2.5x RAM) achieved **7.29 GB/s with Direct I/O vs 0.95 GB/s buffered** — 7.7x improvement. Random reads hit **547K IOPS vs 238K IOPS** buffered, using 80KB of memory versus gigabytes of page cache.

## Buffer model trade-off

Unlike [[monoio]]'s ownership-transfer model, Glommio copies data into internal managed buffers to provide a more traditional borrowing interface. ByteDance's benchmarks attribute Monoio's higher peak network throughput directly to this extra copy overhead.

## Maintenance status

**Effectively unmaintained since Costa's departure.** Open issues accumulate slowly, and a memory corruption bug in the SPSC queue was reported December 2025. Teams considering Glommio should be prepared to fork and maintain it. The GMF framework demonstrated 84% more requests/sec than Tonic-based gRPC, proving the architecture works — the question is long-term viability.

## When to use it

Only if you are building a **storage engine that specifically needs proportional-share scheduling, DMA file abstractions, and latency-class differentiation** — and you have the engineering capacity to maintain a fork. For everything else, [[compio]] is the recommended alternative.
