---
tags:
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# SIMD Programming

Single Instruction, Multiple Data — the hardware capability that consistently separates the fastest data structure implementations from the rest. SIMD processes 16–64 bytes per instruction using vector registers (SSE, AVX2, AVX-512, NEON), transforming sequential comparisons into batch operations.

## Where SIMD dominates data structures

| Structure | SIMD technique | Speedup |
|-----------|---------------|---------|
| [[swiss-table]] hash maps | Parallel H2 fingerprint matching across 16 slots | 3x over `std::unordered_map` |
| [[adaptive-radix-tree]] Node16 | SSE comparison of all 16 keys simultaneously | Enables adaptive node design |
| [[b-tree]] (Algorithmica S+) | Branchless SIMD binary search within nodes | 18x over `std::set` |
| [[binary-fuse-filter\|Blocked Bloom]] | AVX2 checks all hash positions in one cache line | Fastest query throughput |
| [[simdjson]] | AVX2 structural character classification, 32–64 bytes/iter | 4x RapidJSON, 25x nlohmann |
| [[soa-vs-aos\|SoA]] layouts | Direct vector loads from contiguous typed arrays | Up to 12.7x over AoS |

## In Rust

[[pulp]] provides portable SIMD on stable Rust with runtime CPU feature detection and dispatch. The standard library's `std::simd` is still unstable. [[hashbrown]] uses SIMD for Swiss Table probing. [[simdjson]] has Rust bindings via `simd-json`.

## The key insight

SIMD is not just for numerical computing. Its biggest impact is in **metadata scanning** — checking multiple candidates simultaneously in hash maps, trees, and filters. This pattern, pioneered by [[swiss-table]], has become a universal technique for high-performance data structure design.
