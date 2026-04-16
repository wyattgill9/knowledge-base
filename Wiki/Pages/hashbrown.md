---
tags:
  - rust
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
last_updated: 2026-04-15
---

# hashbrown

hashbrown is the Swiss-table-based hash map implementation that underlies `std::HashMap` in Rust. As of version 0.15+, it uses [[foldhash]] as its default hasher, replacing ahash. This means all standard `HashMap` usage benefits from foldhash's performance without any code changes.
