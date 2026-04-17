---
tags:
  - rust
  - concurrency
  - data-structures
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Rust Concurrent Data Structures

An overview of the concurrent maps and channels available beyond `std` in the Rust ecosystem.

## Concurrent HashMaps

| Crate       | Strategy                           | Best for                                             |
| ----------- | ---------------------------------- | ---------------------------------------------------- |
| [[papaya]]  | Lock-free reads, novel reclamation | Read-heavy caches, async-safe, predictable latency   |
| [[dashmap]] | Sharded RwLocks                    | Best write throughput, familiar API, 173M+ downloads |
| [[scc]]     | Aggressive bucket-level locking    | Extreme write contention                             |

**Decision rule:** if your map is read-heavy and used in async code, use papaya. If write-heavy or sync-only, use dashmap. If write contention is extreme, benchmark scc.

## Message passing channels

| Crate | Throughput | Model | Best for |
|---|---|---|---|
| [[kanal]] | 8–16M msg/sec | Unified sync/async | Maximum raw throughput |
| `tokio::sync::mpsc` | Variable | Async MPSC | Within [[tokio]], collocated tasks |
| [[crossbeam-channel]] | High | Sync MPMC | Synchronous multi-producer/multi-consumer |

Within Tokio, `tokio::sync::mpsc` benefits from intra-thread coroutine switching that can outperform kanal when sender and receiver share a worker thread. Outside Tokio or for maximum portable throughput, kanal leads.
