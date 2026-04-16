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

# dashmap

dashmap is the most widely adopted concurrent HashMap in Rust (173M+ downloads), using sharded RwLocks for the best write throughput with a familiar `HashMap`-like API.

## Trade-offs

dashmap's RwLock-based design means reads can block under write contention, and holding a reference across an `.await` point can deadlock in async contexts. For read-heavy caches in async code, [[papaya]] (lock-free reads) is safer. For extreme write contention, [[scc]] uses more aggressive bucket-level locking. dashmap remains the best general-purpose choice for balanced workloads and synchronous code.
