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

# scc

scc (Scalable Concurrent Collections) is a Rust concurrent HashMap using aggressive bucket-level locking, optimized for **extreme write contention** scenarios. For most workloads, [[dashmap]] or [[papaya]] are better choices — scc shines only when write contention is the dominant bottleneck. See [[rust-concurrent-data-structures]].
