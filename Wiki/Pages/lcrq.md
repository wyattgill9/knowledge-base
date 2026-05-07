---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# LCRQ (Linked Concurrent Ring Queue)

LCRQ is the algorithm Morrison and Afek introduced at PPoPP 2013 that reset what fast multi-producer multi-consumer queues look like. It is a linked list of fixed-size concurrent ring buffers (CRQs), each typically holding 4,096 entries; an enqueue or dequeue executes a **single FAA** on a shared index to claim a slot, then a **128-bit CAS** (`CMPXCHG16B` on x86) to synchronize the actual data exchange. Plugged into [[aggregating-funnels|Aggregating Funnels]] (PPoPP 2025), it is the fastest published strict-FIFO MPMC queue.

## Why a single FAA changed everything

Before LCRQ, the canonical lock-free MPMC queue was the [[michael-scott-queue]], which uses a CAS retry loop on the head and tail pointers. Under contention every thread but one fails its CAS and retries — throughput *decreases* as cores rise, the [[faa-vs-cas|contention meltdown]] effect.

LCRQ replaces the contended CAS on a list pointer with an FAA on an integer slot index. FAA always succeeds (`LOCK XADD` is implemented natively in the cache coherence protocol), so producer and consumer each get a unique slot in O(1) hardware time regardless of contention. The data exchange itself uses a 128-bit CAS, but only locally on the claimed slot — which is uncontended in practice.

Published numbers on 4×18-core Intel Xeons:

- **>3× faster** than the [[michael-scott-queue]]
- **>2.5× faster** than flat-combining queues
- The first MPMC queue to scale near-linearly past 32 threads

## How it works

Each CRQ is a ring of `<value, index, safe>` triples. Producers FAA the tail; consumers FAA the head. After claiming slot `i`, a producer attempts CAS to fill an empty slot; a consumer attempts CAS to extract a value matching the expected index. When a CRQ fills up or hits a livelock condition, a new CRQ is appended via Michael-Scott-style CAS on the list spine — but this only happens once per ~4,096 ops, so the amortized cost is negligible.

The "safe" bit prevents a slow producer from writing into a slot that a consumer has already given up on. The threshold mechanism in [[scq|SCQ]] later replaced this with a more memory-efficient scheme.

## The CAS2 problem

LCRQ's reliance on `CMPXCHG16B` made it x86-only for nearly a decade. ARM, PowerPC, RISC-V, and most embedded targets have no equivalent instruction. Two papers solved this without giving up performance:

- **[[scq|SCQ]]** (Nikolaev, DISC 2019) — single-width CAS plus FAA, half the memory footprint, wins outright on PowerPC.
- **[[lprq|LPRQ]]** (Romanov & Koval, PPoPP 2023) — directly modifies the LCRQ algorithm to drop CAS2, matches LCRQ on x86, beats previous portable designs by 1.6×.

## Memory footprint

A naive LCRQ implementation can consume **~400 MB** under load due to per-slot cache-line padding (64 or 128 bytes per `<value, index, safe>` triple to defeat [[false-sharing]]). SCQ cuts this in half by packing entries more densely while preserving the no-false-sharing guarantee on the contended fields.

## Adoption

LCRQ has direct ports in C++ (Pedro Ramalhete's reference), Java, Kotlin, and Go (most via LPRQ). No native Rust port exists as of 2025 — [[crossbeam-array-queue|crossbeam's `ArrayQueue`]] uses a Vyukov-style bounded ring instead. A GPU port reached 201.8 million ops/s at 623 threads on an NVIDIA K20c.

## See also

- [[aggregating-funnels]] — the 2025 layer that gives 2.5× over plain LCRQ
- [[the-fastest-queue]] — full hierarchy
- [[mpmc-queue]] — broader MPMC landscape
- [[faa-vs-cas]] — the underlying primitive choice
