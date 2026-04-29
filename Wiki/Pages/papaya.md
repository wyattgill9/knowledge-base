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

# papaya

A lock-free concurrent HashMap for Rust with novel memory reclamation. The best choice for **read-heavy caches with predictable latency** — reads never block, and it is safe for async usage with no deadlock risk (unlike [[dashmap]]'s RwLock-based design).

## When to use which concurrent map

- **papaya** — lock-free reads, predictable latency, async-safe. Best for read-heavy caches.
- **[[dashmap]]** — sharded RwLocks, best write throughput, familiar HashMap API. 173M+ downloads. Best for balanced read/write workloads.
- **[[scc]]** — aggressive bucket-level locking for extreme write contention.

See [[rust-concurrent-data-structures]] for the full landscape.

## Where Rust concurrent maps sit globally

Even Rust's best concurrent maps trail the C++ state-of-the-art at very high thread counts. CMU's [[parlayhash]] reaches **1,130 Mops at 128 threads** — roughly 7× Meta's `folly::ConcurrentHashMap` and 39× libcuckoo. Papaya, dashmap, and scc do not have published benchmarks at that scale, but they are not architected around epoch-based reclamation in the same way ParlayHash is. For workloads with extreme thread counts where you can use C++, ParlayHash is the headline.

The other caveat: concurrent maps only earn their overhead above ~4–8 threads. Below that, a single-threaded [[hashbrown]] behind a `Mutex` often wins. See [[fastest-hash-map-2025]] for the broader picture.
