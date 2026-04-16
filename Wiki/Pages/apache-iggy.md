---
tags:
  - rust
  - performance
  - architecture
sources:
  - "Raw/Rust/Shard-per-core Rust runtimes - Monoio, Compio, and Glommio compared.md"
last_updated: 2026-04-15
---

# Apache Iggy

Apache Iggy is a message streaming system whose migration from [[tokio]] to [[compio]] (completed December 2025) represents the most thoroughly documented production adoption of any [[thread-per-core]] Rust runtime.

## Results

- **5,000 MB/s throughput** — 5 million messages per second at 1KB each.
- **Sub-millisecond P99 latency** with fsync-per-message persistence.
- **~9.5ms P9999 WebSocket latency.**

## Why Compio

The Iggy team chose [[compio]] over [[monoio]] (citing slower maintenance and [[io-uring]] feature gaps) and [[glommio]] (citing effective abandonment and design disagreements). They reported Compio patches merged "within hours" — a maintenance velocity neither competitor matches. The pluggable driver-executor architecture was critical for [[deterministic-simulation-testing]] of their distributed system behaviors.

## Lessons learned

- **[[mimalloc]] negated Compio's per-operation boxing overhead** — for small, predictable allocations, the cost is dominated by I/O latency.
- **`RefCell` across `.await` is a footgun** — holding a borrow across a yield point allows the executor to yield to another future that borrows the same cell, causing a runtime panic. Iggy adopted ECS-style Struct-of-Arrays decomposition to solve this.
- **Tokio ecosystem incompatibility is real** — they had to create `compio-ws` from scratch because `async-tungstenite` was fundamentally coupled to poll-based I/O.
