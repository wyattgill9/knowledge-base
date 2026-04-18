---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# mimalloc

Microsoft's mimalloc is a compact, high-performance memory allocator that delivers up to **5.3x faster allocation** than glibc malloc under heavy multithreaded workloads, with ~50% less RSS. It excels at allocations <= 4 KB.

## Usage

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

One line in `main.rs` and you're done. This is arguably the single lowest-effort, highest-impact optimization available to any Rust project.

## mimalloc vs jemalloc

Use mimalloc for cross-platform builds and small-allocation-heavy code. Use [[tikv-jemallocator]] for servers with mixed allocation sizes — jemalloc offers more stable latency across all size classes and built-in heap profiling. On musl targets (Alpine/static builds), either allocator is nearly mandatory — musl's default causes a **7x slowdown**.

Cross-language benchmarks confirm mimalloc's dominance: **13% faster** than tcmalloc on the Lean compiler, **15% lower P99 latency** than jemalloc for small allocations. Arena/bump allocators like [[bumpalo]] deliver 2–5x speedup over general malloc for batch allocations. Google's TCMalloc achieves ~6 ns per-CPU-cache allocation at warehouse scale. See [[rust-memory-allocators]] for the full Rust comparison.
