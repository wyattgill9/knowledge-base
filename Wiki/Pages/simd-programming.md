---
tags:
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
  - "Raw/Fastest CS/The fastest dynamic arrays in computer science.md"
last_updated: 2026-05-04
---

# SIMD Programming

Single Instruction, Multiple Data — the hardware capability that consistently separates the fastest data structure implementations from the rest. SIMD processes 16–64 bytes per instruction using vector registers (SSE, AVX2, AVX-512, NEON), transforming sequential comparisons into batch operations.

## Where SIMD dominates data structures

| Structure | SIMD technique | Speedup |
|-----------|---------------|---------|
| [[swiss-table]] hash maps | Parallel H2 fingerprint matching across 16 slots | 3x over `std::unordered_map` |
| [[adaptive-radix-tree]] Node16 | SSE comparison of all 16 keys simultaneously | Enables adaptive node design |
| [[b-tree]] (Algorithmica S+, [[bs-tree]]) | Branchless SIMD search within nodes; AVX-512 16 keys/node in 2 instructions | 7–18× over `std::set` |
| [[fb-plus-tree]] | AVX-512 byte-wise prefix matching for variable-length keys | Closes SIMD gap to tries |
| [[binary-fuse-filter\|Blocked Bloom]] | AVX2 checks all hash positions in one cache line | Fastest query throughput |
| [[simdjson]] | AVX2 structural character classification, 32–64 bytes/iter | 4x RapidJSON, 25x nlohmann |
| [[soa-vs-aos\|SoA]] layouts | Direct vector loads from contiguous typed arrays | Up to 12.7x over AoS |

## In Rust

[[pulp]] provides portable SIMD on stable Rust with runtime CPU feature detection and dispatch. The standard library's `std::simd` is still unstable. [[hashbrown]] uses SIMD for Swiss Table probing. [[simdjson]] has Rust bindings via `simd-json`.

## The key insight

SIMD is not just for numerical computing. Its biggest impact is in **metadata scanning** — checking multiple candidates simultaneously in hash maps, trees, and filters. This pattern, pioneered by [[swiss-table]], has become a universal technique for high-performance data structure design.

The 16-byte `_mm_cmpeq_epi8` is the canonical example. Stored alongside each slot, a 1-byte fingerprint summarizes the actual key; broadcasting the looked-up fingerprint across an SSE register and comparing it against 16 metadata bytes in one instruction filters out ~99.2% of slots without ever touching the key data. [[boost-unordered-flat-map]], [[abseil-flat-hash-map]], [[hashbrown]], [[folly-f14]], and Go 1.24's built-in `map` all rely on variations of this pattern — even Go, which uses portable 64-bit integer SIMD emulation rather than hardware vector instructions. See [[fastest-hash-map-2025]] for the full hash map landscape.

## SIMD memcpy and dynamic-array growth

SIMD's other major contribution to data structures is bulk memory transfer during reallocation. AVX2-optimized memcpy achieves **2–20× faster copying** than scalar code on large aligned blocks, processing 256 bits per instruction; AVX-512 doubles that to 512 bits. For a 1 GB `std::vector<int>` reallocation, the difference between scalar and AVX-512 copying is the difference between memory-bandwidth-bound and instruction-throughput-bound — at the bandwidth limit either way, but reaching it with far fewer instructions issued.

Modern `memcpy`/`memmove` in glibc and musl already use SIMD internally and dispatch to the widest available instruction set at runtime. The remaining lever is **alignment**: passing an aligned destination to `memcpy` lets the implementation use aligned-load instructions, which are faster than unaligned ones on older microarchitectures and avoid cache-line splits everywhere. Crates like `xsimd::aligned_allocator` ensure dynamic arrays produce aligned buffers; Rust's `Vec` is naturally aligned to `std::mem::align_of::<T>()`. See [[fastest-dynamic-arrays]] for where SIMD memcpy fits in the hierarchy of dynamic-array optimizations and [[trivial-relocatability]] for which types qualify for the memcpy path during vector growth.
