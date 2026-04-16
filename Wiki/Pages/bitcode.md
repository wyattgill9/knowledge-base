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

# bitcode

bitcode 0.6 tops the rust_serialization_benchmark in both speed and smallest output size for traditional (non-zero-copy) serialization, beating bincode and postcard. It pairs well with [[rkyv]]: use rkyv for hot-path zero-copy in-memory access, bitcode for wire format and storage where the serialized form needs to be compact and fast to produce.
