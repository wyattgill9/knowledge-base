---
tags:
  - rust
  - performance
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# Rust Memory Allocators

Replacing the default system allocator is one of the lowest-effort, highest-impact optimizations available to any Rust project. On musl targets (Alpine/static builds), it's nearly mandatory — musl's default allocator causes a **7x slowdown**. The cross-language [[fastest-dynamic-arrays]] research makes the case forcefully: switching allocators moves benchmarks more than switching languages does. A `Vec<T>` with [[mimalloc]] outperforms a `Vec<T>` with system malloc by 2–3× on push-heavy workloads, while C, Rust, and C++ vectors with the same allocator land within 10% of each other.

## The two choices

| | [[mimalloc]] | [[tikv-jemallocator]] |
|---|---|---|
| Best for | Cross-platform, small allocs (<= 4 KB) | Servers, mixed allocation sizes |
| Throughput | Up to 5.3x faster than glibc malloc | Slightly lower peak, more consistent |
| Latency | Good | Best — most stable across all sizes |
| Memory | ~50% less RSS than glibc | Comparable to glibc |
| Profiling | External tools only | Built-in via `tikv-jemalloc-ctl` |
| Portability | Excellent (C, compiles everywhere) | Good (C, some platform quirks) |

## Quick start

```rust
// In main.rs — that's it
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

## Arena allocators

For hot paths where you can batch-deallocate, [[bumpalo]] (~2 ns bump allocation) and [[slotmap]] (generational-index arenas) avoid the global allocator entirely. These complement, rather than replace, a global allocator choice.

## Why this matters more than container choice

The [[fastest-dynamic-arrays|dynamic-array research]] established that the allocator dominates almost every other variable. Rust's `Vec<T>` benefits from this implicitly: `Vec`'s reallocation path delegates to the global allocator's `grow`, which on jemalloc/mimalloc/[[tcmalloc]] can perform [[realloc-mremap|in-place expansion]] without copying. The container is the same; the allocator decides whether growth is O(N) or O(1). One `#[global_allocator]` line is, by a wide margin, the highest-leverage optimization in the language.
