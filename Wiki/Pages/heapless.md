---
tags:
  - rust
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# Heapless

heapless provides stack-allocated, fixed-capacity collections (`Vec`, `String`, `HashMap`, queues) using const generics for embedded and `no_std` environments where heap allocation is unavailable. A `heapless::Vec<u8, 64>` stores up to 64 bytes on the stack with no allocator dependency.
