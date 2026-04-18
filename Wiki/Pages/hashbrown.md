---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# hashbrown

hashbrown is a direct [[swiss-table]] port to Rust that became `std::HashMap` since Rust 1.36, delivering **2x speedup** over the prior Robin Hood implementation. As of version 0.15+, it uses [[foldhash]] as its default hasher, replacing ahash. This means all standard `HashMap` usage benefits from foldhash's performance without any code changes.

## Swiss Table mechanics

hashbrown implements the [[swiss-table]] design: each 64-bit hash is split into H1 (group selector) and H2 (7-bit fingerprint). H2 is broadcast across 16 metadata bytes via [[simd-programming|SIMD]] and compared simultaneously — checking 16 slots in ~3 CPU instructions. This technique, pioneered by Google's `absl::flat_hash_map`, now underpins hash maps in C++, Rust, and Go (Go 1.24 adopted Swiss Table as its default runtime map).

In C++, `boost::unordered_flat_map` is the current all-around best performer, and `folly::F14FastMap` (Meta) wins for small value types. See [[swiss-table]] for the full landscape.
