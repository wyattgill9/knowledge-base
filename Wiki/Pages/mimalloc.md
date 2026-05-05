---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# mimalloc

Microsoft's mimalloc is a compact, high-performance memory allocator that delivers up to **5.3x faster allocation** than glibc malloc under heavy multithreaded workloads, with ~50% less RSS. It excels at allocations <= 4 KB. On allocation-heavy workloads, mimalloc delivers a measured **13–22% overall speedup** versus system malloc — making it, alongside [[jemalloc]] and [[tcmalloc]], the highest-leverage one-line performance change available to most projects.

## The allocator dominates the container

The single most important framing the [[fastest-dynamic-arrays]] research established: **switching from glibc to mimalloc moves the needle more than switching from C to Rust**. On a 100M `uint64_t` push_back benchmark, the language ranking (C, Rust, C++) had all entries within ~10% of each other; the allocator swap caused a 2–3× swing. This reorders almost every received opinion about container performance — `std::vector` with mimalloc is typically faster than [[folly-fbvector]] with system malloc, despite the latter being a far more sophisticated container.

## Usage

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

One line in `main.rs` and you're done. This is arguably the single lowest-effort, highest-impact optimization available to any Rust project.

## mimalloc vs jemalloc vs tcmalloc

Use mimalloc for cross-platform builds and small-allocation-heavy code. Use [[jemalloc]] (via [[tikv-jemallocator]] in Rust) for servers with mixed allocation sizes and long-running workloads where fragmentation resistance matters — and when integrating with [[folly-fbvector]] or [[folly-f14]], which depend on jemalloc's `xallocx`. Use [[tcmalloc]] when 4 MB+ allocations dominate; tcmalloc hits 50× system libmalloc throughput at that size. On musl targets (Alpine/static builds), any of the three is nearly mandatory — musl's default causes a **7x slowdown**.

Cross-language benchmarks confirm mimalloc's dominance: **13% faster** than tcmalloc on the Lean compiler, **15% lower P99 latency** than jemalloc for small allocations. Arena/bump allocators like [[bumpalo]] deliver 2–5x speedup over general malloc for batch allocations. See [[rust-memory-allocators]] for the full Rust comparison and [[fastest-dynamic-arrays]] for why this single choice dominates dynamic-array performance.
