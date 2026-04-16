---
tags:
  - rust
  - performance
  - architecture
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# Rust Memory Allocators

Replacing the default system allocator is one of the lowest-effort, highest-impact optimizations available to any Rust project. On musl targets (Alpine/static builds), it's nearly mandatory — musl's default allocator causes a **7x slowdown**.

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
