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

# rapidhash

rapidhash is a non-cryptographic hash function that leads overall throughput benchmarks for Rust hash implementations, with a **4.25 ns geometric mean** across diverse workloads (vs [[foldhash]]'s 4.79 ns).

## When to use it

Use rapidhash for custom hash-based data structures where you control the hasher — specialized hash maps, bloom filters, hash-based deduplication. For standard `HashMap` usage, [[foldhash]] is the path of least resistance since it's now the default hasher for [[hashbrown]] 0.15+ (and by extension, `std::HashMap`).

## The hashing landscape

- **rapidhash** — fastest overall, best for custom structures.
- **[[foldhash]]** — new hashbrown default, faster for integer tuple hashing specifically (0.62 ns vs 0.85 ns), smaller HashMap footprint (40 bytes vs ahash's 64 bytes).
- **ahash** — now legacy, replaced by foldhash in hashbrown.
- **gxhash** — excels at long-string hashing but requires AES hardware and isn't portable.
