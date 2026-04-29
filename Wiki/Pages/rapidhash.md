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

# rapidhash

A non-cryptographic hash function that leads overall throughput benchmarks for Rust hash implementations, with a **4.25 ns geometric mean** across diverse workloads (vs [[foldhash]]'s 4.79 ns). For custom hash-based data structures where you control the hasher, rapidhash is the right default.

## Why hasher choice often dominates

The biggest single performance win in modern hash maps is often the hasher, not the table. Switching from Rust's SipHash default to [[foldhash]] yields **2–5× speedup** in hash-heavy workloads — larger than the gap between [[boost-unordered-flat-map]] and [[abseil-flat-hash-map]]. The compiler `rustc` got a 6% overall speedup from moving to fxhash. Rapidhash takes that idea another step: when you control the hasher in a custom structure, the geometric-mean lead over foldhash compounds across millions of operations. See [[fastest-hash-map-2025]] for context.

## When to use it

Use rapidhash for custom hash-based data structures — specialized hash maps, Bloom filters, hash-based deduplication. For standard `HashMap` usage, [[foldhash]] is the path of least resistance since it is now the default hasher for [[hashbrown]] 0.15+ (and by extension `std::HashMap`).

## The hashing landscape

- **rapidhash** — fastest overall geometric mean; best for custom structures.
- **[[foldhash]]** — current hashbrown default; faster for integer tuple hashing specifically (0.62 ns vs 0.85 ns); smaller `HashMap` footprint (40 bytes vs ahash's 64).
- **fxhash** — small, fast, zero-quality on adversarial input; what `rustc` switched to internally for a 6% overall speedup.
- **gxhash** — excels at long-string hashing; requires AES hardware, not portable.
- **ahash** — legacy; replaced by foldhash in hashbrown.
- **wyhash / xxhash3** — competitive cross-language references.
- **SipHash** — Rust stdlib's safe default; resists hash-flooding but slow. None of the above replace it for adversarial input.

## What "fastest" means here

These hashers all assume non-adversarial input. They produce well-distributed output for typical keys but offer no cryptographic resistance — an attacker who can choose keys can force collisions. For trusted input (internal IDs, parsed tokens, UUIDs) that is fine and the speedup is large. For directly-hashing untrusted external input (HTTP headers, request payloads), keep SipHash or use a randomized seed.
