---
tags:
  - rust
  - concurrency
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Fastest CS/The fastest ordered maps in computer science.md"
last_updated: 2026-05-08
---

# Congee

Rust port of [[art-olc|ART with Optimistic Lock Coupling]]. Hits **150 Mops/sec on 32 cores** with 8-byte integer keys, putting it in the same tier as the C++ reference implementation and far ahead of [[dashmap]] / [[scc]] / [[papaya]] for *ordered* concurrent workloads (those are unordered hash maps and live in a different design space — see [[fastest-hash-map-2025]]).

## Why it exists

The Rust ecosystem has excellent concurrent *unordered* maps but until Congee had no production-quality concurrent *ordered* map matching the C++ state of the art. `BTreeMap` is single-threaded; wrapping it in `RwLock` collapses under contention. Congee fills the same niche [[parlayhash]] fills on the unordered side: the implementation that closes the gap to the global frontier.

## When to reach for it

- Concurrent point operations on integer or fixed-width keys.
- Workloads that need ordered traversal (otherwise prefer [[papaya]] or [[scc]] for read-heavy / write-heavy hash-map paths).
- High thread counts where lock-coupled `BTreeMap` wrappers melt down.

For range-scan-heavy concurrent workloads the global state of the art is [[bp-tree]] (no current Rust port). For string-key high-contention workloads it's [[masstree]] (also unported). Congee is the closest the Rust ecosystem currently sits to the concurrent-ordered-map frontier — see [[rust-concurrent-data-structures]] and [[fastest-ordered-maps]].
