---
tags:
  - cpp
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-29
---

# absl::flat_hash_map

Google's reference [[swiss-table]] implementation, introduced in 2017 and the design that started everything. Still the most widely deployed high-performance hash map in production, powering Google's infrastructure. As of 2025 it is narrowly beaten on most operations by [[boost-unordered-flat-map]], but remains the right choice within Google's ecosystem and for large maps with integer keys — its original sweet spot.

## The original Swiss Table

Each 64-bit hash splits into H1 (slot/group selector) and H2 (a 7-bit fingerprint). H2 is broadcast across 16 metadata bytes via `_mm_set1_epi8` and compared simultaneously with `_mm_cmpeq_epi8`, making one SIMD instruction probe 16 slots. Only on H2 match does the implementation touch the actual key. Elasticsearch reported 6× fewer LLC misses after adopting it; insertions ran ~3× faster than `std::unordered_map` while using 60% less memory.

The header-only clone `phmap::flat_hash_map` (Greg Popovitch) reproduces the design without the Abseil dependency.

## Where it now ranks

In Jackson Allan's May 2024 benchmark, [[boost-unordered-flat-map]] beat it on insert, erase, and lookup-hit. Abseil claws back ground on lookup-miss (1.00× vs Boost's 1.02×) and remains highly competitive overall — but the gap on **erase** (Abseil 2.17× vs Boost 1.36×) reflects Abseil's structural weakness: tombstones.

## The tombstone problem

When a slot is erased, Abseil marks it with a tombstone sentinel rather than reorganizing. Tombstones accumulate under heavy insert/erase cycling and degrade probe-chain performance over time. [[boost-unordered-flat-map]] sidestepped this with an overflow byte; [[ankerl-unordered-dense]] uses Robin Hood backward-shift; [[emhash]] tolerates load factor 0.999 with no tombstones. Abseil is also more sensitive to hash quality than Boost (which auto-bit-mixes weak hashes).

## When it is still the right choice

- Large maps with integer keys (its original benchmark sweet spot).
- Anywhere in Google's C++ ecosystem (it is the reference).
- When you want the most battle-tested production deployment.
- As a known fast default — within ~1.1× of Boost on most operations is still excellent.

## In the broader convergence

The Swiss Table architecture pioneered here now underpins [[hashbrown]] (Rust `std::HashMap` since 1.36), Go 1.24's built-in `map`, and experimentally Java. See [[swiss-table]] for the architecture and [[fastest-hash-map-2025]] for the full landscape.
