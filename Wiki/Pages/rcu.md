---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Read-Copy-Update (RCU)

The optimal concurrency primitive for read-dominated workloads. Linux kernel RCU achieves effectively **zero read-side overhead** — no locks, no atomics, no memory barriers in optimized builds. Readers proceed at full speed as if no synchronization exists; only writers pay coordination costs.

## How it works

RCU's key insight: instead of protecting data with locks, writers create a new version of the data, atomically swap the pointer, and defer freeing the old version until all pre-existing readers have finished. Readers never block, never contend, and never see a partial update.

The "quiescent state" detection — knowing when all readers have finished with old data — is where implementations differ. The Linux kernel uses context switches as quiescent state indicators. Userspace implementations use epoch-based or hazard-pointer-based reclamation.

## Implementations

**Linux kernel RCU** — the reference implementation, powering routing tables, file system caches, and dozens of other kernel subsystems.

**liburcu** — the userspace implementation, powering Knot DNS, GlusterFS, and ISC BIND in production.

**[[crossbeam-epoch|crossbeam-epoch]]** (Rust) — epoch-based reclamation providing similar guarantees in Rust. It underpins most of the Rust concurrent data structure ecosystem, with crossbeam-deque alone having 299M+ downloads. [[papaya]], [[dashmap]], and [[scc]] all build on epoch-based or similar reclamation.

## When to use RCU vs other approaches

RCU is ideal when reads vastly outnumber writes (10:1 or more). For balanced read/write workloads, sharded locks ([[dashmap]]) or lock-free queues ([[lmax-disruptor]]) may be better. For write-heavy concurrent maps, [[scc]] is optimized for extreme write contention.

The [[thread-per-core]] architecture takes a different approach entirely — eliminating shared state so no synchronization is needed at all, at the cost of data partitioning complexity.
