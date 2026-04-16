---
tags:
  - rust
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# rkyv

rkyv (archive) is Rust's zero-copy deserialization framework — it writes data in a format that can be accessed directly from the serialized bytes without any deserialization step. Version 0.8 shipped September 2024 and remains uncontested in its niche.

## Performance

~21 ns access time vs 300+ ns for traditional deserialization (serde-based formats). This isn't a percentage improvement — it's an order-of-magnitude difference. The trade-off is that archived data uses a fixed format (default little-endian in 0.8) and requires the rkyv dependency to read.

## What 0.8 changed

- **Rancor-based error handling** — structured, composable errors replacing the old `Unreachable` panics.
- **Safe validation API** — validate archived data before accessing it, closing the previous unsoundness window.
- **Default little-endian format** — standardized byte order.

## Complementary crates

rkyv excels at zero-copy access but isn't optimized for traditional serialize-then-deserialize workflows. For those, [[bitcode]] 0.6 now tops the rust_serialization_benchmark in both speed and smallest output size, beating bincode and postcard. The two crates pair well: rkyv for hot-path in-memory access, bitcode for wire format and storage.
