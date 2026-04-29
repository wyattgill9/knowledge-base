---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# hashbrown

A direct [[swiss-table]] port to Rust that became `std::HashMap` since Rust 1.36 (2019), delivering **2× speedup** over the prior Robin Hood implementation. As of version 0.15+, it uses [[foldhash]] as its default hasher, replacing ahash. All standard `HashMap` usage benefits from foldhash's performance with no code changes. With foldhash in place, hashbrown tracks [[abseil-flat-hash-map]] to within ~5% — putting Rust's stdlib map in the same league as Google's reference.

## Swiss Table mechanics

hashbrown implements the [[swiss-table]] design: each 64-bit hash splits into H1 (group selector) and H2 (7-bit fingerprint). H2 is broadcast across 16 metadata bytes via [[simd-programming|SIMD]] and compared simultaneously — checking 16 slots in ~3 CPU instructions. The technique, pioneered by Google's [[abseil-flat-hash-map|`absl::flat_hash_map`]], now underpins hash maps across C++, Rust, and Go (Go 1.24 adopted Swiss Table as its default runtime map: 31% faster inserts, 21% faster lookups, 70% memory reduction at Datadog).

In the C++ family, [[boost-unordered-flat-map]] is the current 2025 all-around best performer (overflow byte avoids Abseil's tombstone rot), and [[folly-f14]] wins on memory footprint. See [[swiss-table]] and [[fastest-hash-map-2025]] for the full landscape.

## The hash function dominates

The most impactful change to Rust hash performance in recent years was not a table change. Switching from SipHash (the safe `std::HashMap` default for hash-flooding resistance) to [[foldhash]] yields **2–5× speedup** in hash-heavy workloads. The compiler `rustc` itself saw a **6% overall speedup** from switching to fxhash. Hashbrown ships with foldhash by default; if you control the hasher in a custom hash structure, [[rapidhash]] leads overall throughput at 4.25 ns geometric mean.

## Public crate vs std

The `hashbrown` crate exists for users who want a faster `HashMap` than `std`'s default DDoS-resistant SipHash configuration, or who need newer hashbrown features ahead of the stdlib re-export. Most code should just use `std::collections::HashMap` — it *is* hashbrown. Reach for the crate only when you need a non-default hasher (e.g., foldhash explicitly) or hashbrown-specific raw entry APIs.

## When to step outside hashbrown

For concurrent workloads, [[papaya]] (lock-free reads), [[dashmap]] (sharded RwLock), and [[scc]] (extreme write contention) cover Rust's options — but if you can use C++, [[parlayhash]] hits 1,130 Mops at 128 threads, an order of magnitude past anything Rust currently fields. For static read-only key sets, the `phf` crate provides compile-time [[perfect-hashing]].
