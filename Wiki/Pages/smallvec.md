---
tags:
  - rust
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Rust/What A+ Rust design actually looks like.md"
last_updated: 2026-04-16
---

# SmallVec

SmallVec stores up to N elements inline on the stack before falling back to heap allocation. Ideal for collections that are usually small (3–8 elements) but occasionally grow — avoids heap allocation entirely on the common path. Part of the [[rust-allocation-patterns|borrow → Cow → owned]] allocation hierarchy for expert Rust performance.
