---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# Swiss Table

The hash map design that changed everything. Introduced by Google in 2017, Swiss Table uses SIMD-parallel metadata probing to check 16 slots in approximately 3 CPU instructions — a technique so effective that it now underpins hash maps in C++ ([[abseil-flat-hash-map|Abseil]], [[boost-unordered-flat-map|Boost]], [[folly-f14|folly F14]]), Rust ([[hashbrown]] in `std` since 1.36), Go (built-in `map` since 1.24), and experimentally Java. The convergence on Swiss Table-derived designs is the defining hash map trend of 2018–2025.

## How it works

Every slot has a 1-byte metadata entry. Each 64-bit hash splits into **H1** (slot/group selector) and **H2** (a 7-bit fingerprint stored as metadata). On lookup, H1 selects the group, then H2 is broadcast across 16 metadata bytes via `_mm_set1_epi8` and compared simultaneously with `_mm_cmpeq_epi8`. This checks **16 slots in ~3 CPU instructions**. Only on H2 match does the implementation touch the actual key.

Two reinforcing properties make this work:

- **Metadata acts as a Bloom filter.** With 7 bits of fingerprint per slot, false positives occur with ~1/128 probability per slot — meaning ~99.2% of lookups resolve without ever comparing the actual key.
- **Power-of-2 sizing.** A bitwise AND replaces ~20–40-cycle modulo. Combined with a high-entropy hash (wyhash, foldhash, Abseil hash), this is strictly superior to prime-sized tables.

Group probing also unlocks a much higher load factor than traditional open addressing — Boost and Abseil both fix it at **0.875**, where collision costs are amortized across the SIMD-compared group.

## Boost vs Abseil: the overflow byte vs tombstone divide

Both [[boost-unordered-flat-map]] and [[abseil-flat-hash-map]] are Swiss Table variants, but they handle deletion differently — and that single decision drives much of the 2025 performance gap.

**Abseil uses tombstones.** A deleted slot is marked with a sentinel; tombstones accumulate over time and degrade probe-chain performance under heavy insert/erase cycling. This shows up sharply on erase benchmarks (Abseil 2.17× the leader vs Boost 1.36× in Jackson Allan's suite).

**Boost uses an overflow byte per group.** It records whether *any* overflow has ever occurred in that group. Lookups can terminate at non-overflowed groups instantly. The author measured 3.2× fewer comparisons on unsuccessful lookups at high load factors.

Boost also stores a slightly richer fingerprint (7.99 bits vs Abseil's 7) and auto-bit-mixes weak hashes. As of 2025 it is the consensus general-purpose winner; Abseil remains the production reference and excels for large maps with integer keys. See [[fastest-hash-map-2025]] for the full benchmark picture.

## Implementations

| Implementation | Distinguishing property |
|---|---|
| [[boost-unordered-flat-map]] | 2025 general-purpose winner; overflow byte; auto bit-mixing |
| [[abseil-flat-hash-map]] | Original Swiss Table; reference at Google scale |
| [[hashbrown]] | Direct port to Rust; `std::HashMap` since 1.36; uses [[foldhash]] |
| [[folly-f14|folly F14]] | 14-entry chunks; lowest memory footprint (24.7 B/elt); overflow counting |
| Go 1.24 `map` | 8-slot groups; portable 64-bit integer SIMD emulation |

The header-only clone `phmap::flat_hash_map` reproduces Abseil's design without the Abseil dependency.

## When Swiss Table loses

For **iteration-heavy workloads**, [[ankerl-unordered-dense]] stores all key-value pairs in a contiguous `std::vector`, making iteration as fast as scanning an array — 2.55× faster than Abseil. For **extreme load factors**, [[emhash]] operates at 0.999 with 3-way linear probing and no tombstones. For **lowest memory footprint**, [[folly-f14]]'s 14-entry chunks. For **read-only static data**, [[perfect-hashing]] beats every probe-based design because it eliminates probing entirely.

For **concurrent workloads**, see [[parlayhash]] (CMU, 1,130 Mops at 128 threads — an order of magnitude past everything else), and in Rust [[papaya]], [[dashmap]], [[scc]].

## The hash function caveat

A Swiss Table is only as fast as its hasher. Switching from SipHash to [[foldhash]] in Rust delivered 2–5× speedup in hash-heavy workloads — *larger than any difference between table implementations*. The compiler `rustc` itself got 6% overall from switching to fxhash. For specialized workloads where you control the hasher, [[rapidhash]] leads overall throughput. The "fastest hash map" question often resolves to the hash function, not the table.

## Why it matters

Swiss Table proved that [[simd-programming|SIMD]] is not just for numerical computing — it fundamentally changes how core data structures are designed. The same SIMD-parallel scanning technique now appears in [[adaptive-radix-tree]] (Node16), [[binary-fuse-filter|blocked Bloom filters]], and [[b-tree]] node search. And in 2025, [[elastic-hashing]] disproved Yao's 1985 conjecture on probe complexity, suggesting the design space is not yet exhausted even within greedy-insertion open addressing.
