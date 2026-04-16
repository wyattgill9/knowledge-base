---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# bytes

The bytes crate provides `Bytes` and `BytesMut` — `Arc`-backed byte buffers that enable zero-copy sharing throughout the [[tokio]] ecosystem. Used by [[hyper]], [[tonic]], and every major Rust networking crate.

## Why it matters

Passing `Bytes` between tasks is a reference-count increment, not a copy. `BytesMut` provides exclusive mutable access that can be frozen into a shared `Bytes` when writing is done. This is the standard pattern for buffer management in async Rust networking.
