---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
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

See [[rust-memory-allocators]] for the full comparison.
