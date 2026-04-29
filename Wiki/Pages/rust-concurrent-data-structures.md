---
tags:
  - rust
  - concurrency
  - data-structures
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
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

**Where Rust sits globally.** Even Rust's best concurrent maps trail the C++ state-of-the-art at very high thread counts. CMU's [[parlayhash]] hits 1,130 Mops at 128 threads via epoch-based reclamation — roughly 7× Meta's `folly::ConcurrentHashMap` and 39× libcuckoo. None of papaya, dashmap, or scc has been benchmarked at that scale, and none is architected the same way. For services that genuinely run at 64+ threads with hot map access and can step outside Rust, ParlayHash is the headline. For the 4–32 thread range where most Rust services live, the table above stands. See [[fastest-hash-map-2025]] for the broader picture.

## Message passing channels

| Crate | Throughput | Model | Best for |
|---|---|---|---|
| [[kanal]] | 8–16M msg/sec | Unified sync/async | Maximum raw throughput |
| `tokio::sync::mpsc` | Variable | Async MPSC | Within [[tokio]], collocated tasks |
| [[crossbeam-channel]] | High | Sync MPMC | Synchronous multi-producer/multi-consumer |

Within Tokio, `tokio::sync::mpsc` benefits from intra-thread coroutine switching that can outperform kanal when sender and receiver share a worker thread. Outside Tokio or for maximum portable throughput, kanal leads.
