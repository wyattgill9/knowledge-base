---
tags:
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-05-04
---

# jemalloc

jemalloc is a general-purpose malloc replacement originally developed for FreeBSD and adopted by Facebook for production use. Its defining strengths are excellent fragmentation resistance under long-running workloads and a richer allocator API than POSIX malloc — most notably `xallocx()`, which enables in-place expansion that [[folly-fbvector]] uses to skip copies during vector growth. jemalloc delivers approximately **30% higher throughput at 36 threads** under high contention compared to system malloc.

## The xallocx API

`xallocx(ptr, size, extra, flags)` is jemalloc's "try to extend" primitive. It returns the actual allocated size if jemalloc can grow the existing allocation in place, or the original size if it cannot. This is the user-space counterpart to [[realloc-mremap|`mremap`]] — it lets the container ask "can I grow without copying?" and act on the answer.

[[folly-fbvector]] is built around this call: every reallocation first attempts `xallocx`, falling back to a fresh allocation only when in-place expansion fails. Testing with 256M floats showed `xallocx` avoided copies for most growth attempts on buffers above 32KB. `std::vector` cannot use this because `std::allocator` exposes no equivalent capability.

## Size classes and goodMallocSize

jemalloc partitions allocations into size classes — each request is rounded up to a class boundary. `goodMallocSize(n)` returns the allocator's preferred size at or above `n`, allowing containers to query "what would you actually allocate?" and use that capacity instead of wasting it. fbvector uses this to size its growth requests so no allocation slack is left unused.

The size-class structure also explains why jemalloc-aware vectors can use a 2× growth factor for small buffers without waste — small size classes are typically themselves powers of two, so doubling matches the class boundaries exactly. See [[growth-factor-analysis]] for why this matters less than the allocator dominance argument suggests.

## The Rust wrapper

`tikv-jemallocator` is the maintained Rust wrapper, providing the most stable allocation latency across all size classes and built-in heap profiling via `tikv-jemalloc-ctl`. See [[tikv-jemallocator]] for the Rust-specific entry point and [[rust-memory-allocators]] for the choice between mimalloc and jemalloc in Rust.

## Positioning vs alternatives

| Allocator | Strength | Weakness |
|-----------|----------|----------|
| jemalloc | Fragmentation resistance under long-running mixed workloads; rich API | Larger memory overhead on quiet workloads |
| [[mimalloc]] | Lowest small-allocation latency, lowest RSS | Less mature API surface |
| [[tcmalloc]] | 50× throughput on large allocations; per-CPU caching | More invasive integration |
| glibc malloc | Universal availability | 7× slower than jemalloc on some workloads; weak under contention |

The general lesson — dramatically supported by the [[fastest-dynamic-arrays|dynamic-array benchmarks]] — is that switching from system malloc to *any* of jemalloc, mimalloc, or tcmalloc is the single highest-leverage performance change available. Linking choice often matters more than container choice.

## Where jemalloc is the right pick

- Long-running services (databases, message brokers) where fragmentation resistance compounds over hours and days.
- High-contention workloads with many threads doing mixed-size allocations.
- C++ projects committing to [[folly-fbvector]] or [[folly-f14]], which depend on jemalloc-specific calls.
- Server workloads where consistent latency matters more than peak throughput.

See [[fastest-dynamic-arrays]] for the central role jemalloc plays in the dynamic-array hierarchy.
