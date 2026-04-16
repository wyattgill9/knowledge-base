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

# foldhash

foldhash is the default hasher for [[hashbrown]] 0.15+, replacing ahash — a significant ecosystem shift since hashbrown underlies `std::HashMap`. It delivers better performance than ahash with a smaller HashMap footprint (40 bytes vs 64 bytes).

## Performance profile

foldhash is faster than ahash across the board and faster than [[rapidhash]] for integer tuple hashing specifically (0.62 ns vs 0.85 ns). However, rapidhash still leads the overall geometric mean benchmark (4.25 ns vs 4.79 ns). For standard `HashMap` usage where you don't control the hasher, foldhash is what you get — and it's good.

## The ahash succession

ahash was the previous hashbrown default and accumulated massive adoption. foldhash replaced it by being both faster and more memory-efficient. ahash should be considered legacy for new projects.
