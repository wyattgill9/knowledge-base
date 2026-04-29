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

# dashmap

The most widely adopted concurrent HashMap in Rust (173M+ downloads), using sharded RwLocks for the best write throughput in the Rust ecosystem with a familiar `HashMap`-like API.

## Trade-offs

dashmap's RwLock-based design means reads can block under write contention, and holding a reference across an `.await` point can deadlock in async contexts. For read-heavy caches in async code, [[papaya]] (lock-free reads) is safer. For extreme write contention, [[scc]] uses more aggressive bucket-level locking. dashmap remains the best general-purpose choice for balanced workloads and synchronous code.

## How it scales

Sharded locking keeps contention bounded as long as your access pattern is well-distributed across keys. A hot key concentrated on one shard becomes a serialization point; that is the structural limit of every shard-locked design.

For comparison at extreme thread counts: [[parlayhash]] (CMU C++) hits 1,130 Mops at 128 threads via epoch-based reclamation rather than locking, an order of magnitude past anything in Rust. Rust's concurrent map ecosystem (dashmap, papaya, scc) is not architected to match that asymptote — though for the 4–32 thread range where most Rust services live, dashmap is competitive and the API is more ergonomic. See [[fastest-hash-map-2025]] for the global landscape.
