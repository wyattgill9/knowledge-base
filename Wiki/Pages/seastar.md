---
tags:
  - cpp
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Seastar

Seastar is a C++ framework built by Avi Kivity's team at ScyllaDB that proved the [[thread-per-core]] model at scale: pin one thread per physical core, give each thread its own memory allocator, network queue, and storage partition, and communicate only through explicit message passing. ScyllaDB claims near-linear scaling — doubling cores doubles throughput — because there is essentially no Amdahl's Law serial bottleneck from inter-core coordination.

[[glommio]] is Seastar's direct spiritual successor in Rust, designed by Glauber Costa after seven years building ScyllaDB. ByteDance independently arrived at the same conclusions when building [[monoio]] for their proxy infrastructure.
