---
tags:
  - rust
  - architecture
  - concurrency
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Deterministic Simulation Testing

Deterministic Simulation Testing (DST) is a testing methodology for distributed systems where the time wheel, scheduler, and I/O driver are replaced with deterministic mocks, enabling fully reproducible execution of concurrent system behaviors. FoundationDB pioneered this approach, and it has become a gold standard for verifying distributed system correctness.

## In the Rust ecosystem

[[compio]]'s pluggable driver-executor architecture is the only Rust [[thread-per-core]] runtime that supports DST. Because the I/O driver is decoupled from the executor, test harnesses can inject deterministic time, network partitions, and disk failures while replaying the exact same execution sequence. [[apache-iggy]] cited this capability as critical to their choice of Compio over [[monoio]] and [[glommio]], whose drivers and executors are tightly coupled.
