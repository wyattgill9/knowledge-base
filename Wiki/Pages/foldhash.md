---
tags:
  - rust
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# foldhash

The default hasher for [[hashbrown]] 0.15+, replacing ahash — a quietly significant ecosystem shift since hashbrown underlies `std::HashMap`. foldhash is faster than ahash across the board with a smaller `HashMap` footprint (40 bytes vs 64). With foldhash in place, Rust's stdlib map matches [[abseil-flat-hash-map]] within ~5%, closing a longstanding gap to C++.

## Why the hasher choice matters more than the table

The deepest insight in 2025's hash map landscape is that **hash function choice often dominates table choice**. Switching from SipHash (Rust's safe default for hash-flooding resistance) to foldhash yields **2–5× speedup** in hash-heavy workloads — *larger than any difference between Swiss Table implementations*. The Rust compiler itself got a **6% overall speedup** from moving to fxhash. By comparison, [[boost-unordered-flat-map]] beats [[abseil-flat-hash-map]] by ~10–30% on most operations.

This recontextualizes "fastest hash map" benchmarks. For workloads dominated by hashing time (short keys, simple types), the hasher swamps the table. See [[fastest-hash-map-2025]] for the full picture.

## Performance profile

foldhash is faster than ahash across the board and faster than [[rapidhash]] for integer tuple hashing specifically (0.62 ns vs 0.85 ns). However, rapidhash still leads the overall geometric mean benchmark (4.25 ns vs 4.79 ns). For standard `HashMap` usage where you don't control the hasher, foldhash is what you get — and it is good.

## The ahash succession

ahash was the previous hashbrown default and accumulated massive adoption. foldhash replaced it by being both faster and more memory-efficient. ahash should be considered legacy for new projects.

## When to pick something else

- **rapidhash** — fastest overall geometric mean; right for custom hash structures where you control the hasher.
- **gxhash** — excels at long-string hashing but requires AES hardware and isn't portable.
- **SipHash** — keep when adversarial input is a real threat (e.g., user-controlled keys directly hashed). foldhash is not a hash-flooding-resistant hasher in the cryptographic sense.

For more on the broader hashing landscape, see [[rapidhash]].
