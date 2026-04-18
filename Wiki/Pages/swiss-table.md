---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Swiss Table

The hash map design that changed everything. Introduced by Google in 2017, Swiss Table uses SIMD-parallel metadata probing to check 16 slots in approximately 3 CPU instructions — a technique so effective that it now underpins hash maps in C++, Rust, and Go.

## How it works

Every slot has a 1-byte metadata entry. Each 64-bit hash is split into H1 (57-bit group selector) and H2 (7-bit fingerprint). On lookup, H1 selects the group, then H2 is broadcast across 16 metadata bytes via `_mm_set1_epi8()` and compared simultaneously with `_mm_cmpeq_epi8()`. This checks **16 slots in ~3 CPU instructions**. Only when an H2 match is found does the implementation check the actual key.

## Implementations

**`absl::flat_hash_map`** (Google Abseil) is the original production implementation. Elasticsearch reported 6x fewer LLC misses after adopting it, and it runs approximately 3x faster than `std::unordered_map` for insertions while using 60% less memory.

**`boost::unordered_flat_map`** (Joaquín M López Muñoz) is now the **all-around best C++ performer** according to Jackson Allan's 2024 benchmarks. It uses 15-element groups with an overflow byte instead of tombstones, meaning performance never degrades under heavy insert/erase workloads — a weakness of the original Swiss Table. It automatically applies bit-mixing for poor hash functions.

**[[hashbrown]]** is a direct Swiss Table port to Rust that became the standard library `HashMap` since Rust 1.36, delivering 2x speedup over the prior Robin Hood implementation. It now uses [[foldhash]] as its default hasher.

**`folly::F14FastMap`** (Meta) takes a different approach: 14-entry chunks co-locating metadata and data, better for small value types (sizeof < 24). Nathan Bronson reported F14 beating Swiss Table on some of Google's own benchmarks.

**Go 1.24** adopted Swiss Table as its default runtime map implementation.

## When Swiss Table loses

For iteration-heavy workloads, `ankerl::unordered_dense::map` stores all key-value pairs in a contiguous `std::vector`, making iteration as fast as scanning an array. For extreme load factors, `emhash` operates at load factor 0.999 using 3-way linear probing with x86 bit-scan instructions.

For concurrent workloads, see [[concurrent-queues|concurrent data structures]] — [[papaya]], [[dashmap]], and [[scc]] serve different concurrency profiles. **ParlayHash** (CMU) achieves 1,130 million ops/sec at 128 threads, an order of magnitude faster than libcuckoo and Intel TBB.

## Why it matters

Swiss Table proved that [[simd-programming|SIMD]] is not just for numerical computing — it fundamentally changes how we design core data structures. The same SIMD-parallel scanning technique now appears in [[adaptive-radix-tree]] (Node16), [[binary-fuse-filter|blocked Bloom filters]], and [[b-tree]] node search.
