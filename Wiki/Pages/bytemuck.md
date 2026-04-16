---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# bytemuck

bytemuck provides safe casting between plain-data types (`Pod`, `Zeroable`) in Rust. It's widely used in the Solana and wgpu ecosystems for GPU buffer management and on-chain data layout.

## Current status

As of 2026, [[zerocopy]] 0.8 is the recommended alternative for new code. zerocopy offers formally verified safety (Kani proofs), enum support, DST handling, `UnsafeCell`/atomics awareness, and built-in byte-order types — features bytemuck lacks. However, bytemuck's simpler `Pod`-based API and deep ecosystem integration mean migration of existing code isn't urgent.
