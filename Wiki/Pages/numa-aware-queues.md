---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# NUMA-Aware Queues

On multi-socket systems, remote memory access adds significant latency from interconnect traversal — a cache line bouncing across the QPI/UPI/Infinity Fabric link can cost 200–400 ns versus 50–100 ns within a socket. A NUMA-oblivious lock-free queue ends up paying this cost on most operations, because contention naturally spreads the queue's hot lines across all sockets. NUMA-aware queue designs route operations through per-node delegates to keep cache lines on the local interconnect.

## NUMA Node Delegation (Nuddle)

The standard pattern, generalized as **NUMA Node Delegation (Nuddle)**, transforms a NUMA-oblivious data structure into a NUMA-aware one without changing its core algorithm:

1. One dedicated **server thread per NUMA node** owns the local part of the structure.
2. **Client threads** on each node enqueue/dequeue requests via a local [[spsc-queue|SPSC]] queue to their server.
3. Servers communicate across NUMA boundaries only when strictly necessary — typically to balance the queue between nodes.

Cross-socket cache traffic drops to once per batch instead of once per operation. The trade-off is added latency — every client operation now goes through a delegate — but on a contended workload the latency was already much higher anyway.

## Empirical results

- **SmartPQ** (Giannoula et al., 2024) — NUMA-aware concurrent priority queue achieving **1.87× speedup** over the best NUMA-oblivious priority queue at 2-socket and 4-socket scales.
- **Linux kernel NUMA-aware qspinlock** (merged ~2021) — splits MCS wait queues by NUMA node, approaching **2× speedup** under heavy contention. Same idea applied to spinlocks rather than queues.
- **NUMA-aware [[lmax-disruptor|Disruptor]]** experiments — sharding the ring buffer per NUMA node and treating cross-shard hops as a separate stage gives near-linear scaling on 2-socket systems.

## When it matters

NUMA-awareness only helps when:

- The system is multi-socket (single-socket NUMA via chiplets is typically negligible).
- The queue is contended — uncontended queues never bounce cache lines anyway.
- Threads are pinned to NUMA nodes — without pinning, the OS scheduler will move threads around and break the locality assumption.

For small servers (single socket, ≤32 threads), [[lcrq|LCRQ]] / [[scq|SCQ]] without NUMA awareness is fine. For 2+ socket systems running contended workloads, the 1.5–2× available from NUMA delegation is one of the largest remaining knobs after FAA-vs-CAS and 128-byte padding.

## See also

- [[lcrq]], [[scq]] — the queues NUMA delegation typically wraps
- [[cache-coherency]] — why cross-socket traffic costs what it costs
- [[thread-per-core]] — the architectural extreme that pushes this idea everywhere
- [[the-fastest-queue]] — full hierarchy
