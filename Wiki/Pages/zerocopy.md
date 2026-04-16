---
tags:
  - rust
  - performance
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# zerocopy

Google's zerocopy crate provides safe, zero-cost conversions between byte sequences and typed data. Version 0.8 was a landmark release that makes it the recommended choice over [[bytemuck]] for all new Rust code requiring transmute-like operations.

## What 0.8 brought

- **`TryFromBytes`** for validated conversions — parse bytes into types with runtime validation, not just `FromBytes` blanket trust.
- **Full enum support** with repr discriminant handling.
- **Slice DST support** — types like `[Header, [Item]]` that bytemuck cannot express.
- **`UnsafeCell` and atomics handling** — correct treatment of interior mutability.
- **Built-in byte-order types** — no separate endianness crate needed.
- **Formal verification with Kani proofs** — the safety guarantees are machine-checked, not just "well-tested."

## zerocopy vs bytemuck

For network protocol parsing, security-critical paths, and complex data structures (enums, DSTs), zerocopy 0.8 is strictly superior. [[bytemuck]] remains simpler for basic `Pod` casting and has deep roots in the Solana/wgpu ecosystem, so migration isn't urgent for existing code. But for anything new, use zerocopy.

zerocopy is developed in coordination with Rust's Project Safe Transmute, making it the most likely crate to align with future language-level transmute safety guarantees.
