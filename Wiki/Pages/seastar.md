---
tags:
  - cpp
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Seastar

Seastar is a C++ framework built by Avi Kivity's team at ScyllaDB that proved the [[thread-per-core]] model at scale: pin one thread per physical core, give each thread its own memory allocator, network queue, and storage partition, and communicate only through explicit message passing. ScyllaDB claims near-linear scaling — doubling cores doubles throughput — because there is essentially no Amdahl's Law serial bottleneck from inter-core coordination.

[[glommio]] is Seastar's direct spiritual successor in Rust, designed by Glauber Costa after seven years building ScyllaDB. ByteDance independently arrived at the same conclusions when building [[monoio]] for their proxy infrastructure.

Seastar's own concurrency model — futures, continuations, sharded executors — was the design that informed P2300 and ultimately C++26 [[senders-receivers]]. The standard's value-flow asynchronous model is essentially Seastar's continuation-style futures refined into a scheduler-agnostic algebra. Where Seastar baked in its own scheduler, `std::execution` makes the scheduler a parameter — but the structural lineage from Seastar through libunifex to P2300 is direct.
