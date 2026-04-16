---
tags:
  - rust
  - concurrency
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# papaya

papaya is a lock-free concurrent HashMap for Rust with novel memory reclamation. It's the best choice for **read-heavy caches with predictable latency** — reads never block, and it's safe for async usage with no deadlock risk (unlike [[dashmap]]'s RwLock-based design).

## When to use which concurrent map

- **papaya** — lock-free reads, predictable latency, async-safe. Best for read-heavy caches.
- **[[dashmap]]** — sharded RwLocks, best write throughput, familiar HashMap API. 173M+ downloads. Best for balanced read/write workloads.
- **[[scc]]** — aggressive bucket-level locking for extreme write contention.

See [[rust-concurrent-data-structures]] for the full landscape including channels and queues.
