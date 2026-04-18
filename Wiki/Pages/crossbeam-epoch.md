---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# crossbeam-epoch

Epoch-based memory reclamation for lock-free data structures in Rust. Provides the Rust ecosystem's equivalent of [[rcu|Read-Copy-Update]] — allowing readers to access shared data without locks while writers safely defer deallocation of old versions.

crossbeam-epoch underpins most of the Rust concurrent data structure ecosystem, with crossbeam-deque alone having 299M+ downloads. [[papaya]], [[scc]], and other lock-free Rust structures build on epoch-based or similar reclamation schemes.
