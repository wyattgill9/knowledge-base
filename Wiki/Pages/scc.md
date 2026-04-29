---
tags:
  - rust
  - concurrency
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# scc

scc (Scalable Concurrent Collections) is a Rust concurrent HashMap using aggressive bucket-level locking, optimized for **extreme write contention** scenarios. For most workloads, [[dashmap]] or [[papaya]] are better choices — scc shines only when write contention is the dominant bottleneck.

For very high thread counts where you can step outside Rust, CMU's [[parlayhash]] reaches 1,130 Mops at 128 threads via epoch-based reclamation, an order of magnitude past lock-based designs. See [[rust-concurrent-data-structures]] and [[fastest-hash-map-2025]].
