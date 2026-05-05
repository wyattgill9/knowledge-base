---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# tikv-jemallocator

tikv-jemallocator is the maintained Rust wrapper for [[jemalloc]], providing the most stable allocation latency across all size classes and thread counts. It includes built-in heap profiling via `tikv-jemalloc-ctl`. The underlying jemalloc allocator delivers approximately 30% higher throughput at high contention vs system malloc, and its `xallocx` API is the user-space counterpart to [[realloc-mremap|`mremap`]] — letting containers like [[folly-fbvector]] grow in place without copying. tikv-jemallocator inherits these capabilities transparently for any Rust `Vec<T>` since `Vec`'s reallocation goes through the global allocator's grow path.

## When to choose jemalloc

jemalloc is the better choice for **servers with mixed allocation sizes** — its arena-based design handles the full spectrum from tiny allocations to large mmap'd regions with consistent latency. The built-in profiling (`tikv-jemalloc-ctl`) lets you inspect heap state without external tools. For long-running workloads where fragmentation accumulates, jemalloc's resistance is materially better than glibc's.

For cross-platform builds or workloads dominated by small allocations (<= 4 KB), [[mimalloc]] may be faster. On musl targets, either is nearly mandatory. See [[rust-memory-allocators]] for the full picture and [[fastest-dynamic-arrays]] for why the allocator choice often matters more than the container choice.
