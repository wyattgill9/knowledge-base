---
tags:
  - concurrency
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Slab List

Ashkiani et al., IPDPS 2018. A linked list redesigned for GPU warp execution: each "slab" fills a **warp-width unit (32 threads)** and is processed cooperatively, enabling coalesced memory access. **512 million updates/second** and **937 million search queries/second** on an NVIDIA Tesla K40c — performance unreachable by any CPU-shaped linked list.

## Why warps change the design

A GPU warp executes 32 threads in lockstep on a single instruction. If those 32 threads chase 32 independent linked-list pointers, the warp issues 32 uncoalesced memory accesses — the worst case for GPU memory throughput. If instead the 32 threads cooperate to traverse a single list, accessing 32 contiguous bytes in one transaction, the warp gets one coalesced load.

Slab Lists structure each linked-list "node" as a **slab** sized to one warp-width: 32 elements + a single `next` pointer. The 32 threads in a warp each load one element of the slab in parallel; a ballot/vote operation across the warp resolves which thread holds the matching key; the next slab pointer is followed cooperatively. The result is a linked structure whose traversal pattern is exactly what coalesced memory throughput rewards.

The same warp-cooperative redesign principle produces the [[gpu-queues|fastest GPU queues]] like [[bacq]] (Boundary-Aware Concurrent Queue, 2024), where ignoring warp execution causes [[faa-vs-cas|CAS]] to **melt down 1,112× at high thread counts**. Warp-aware design is mandatory on GPU; nothing else even competes.

## Where it sits

Slab Lists are the **GPU member** of the [[fastest-linked-lists|linked-list hierarchy]] and a clean example of the deeper principle running through the whole landscape: the fastest "linked lists" are the ones that gave up textbook linked-list memory layout in favor of whatever the underlying hardware actually rewards. CPUs reward [[plf-list|block allocation]] and [[vector-backed-list|contiguous index-based linking]]; GPUs reward warp-aligned slabs. The structures share a name and very little else.
