---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Aggregating Funnels

Aggregating Funnels (Roh, Wei, Ruppert et al., **PPoPP 2025**) is a software-combining layer that replaces a hardware FAA hotspot with batched FAA operations across threads. Plugged into [[lcrq|LCRQ]], it improves throughput by **up to 2.5× at high thread counts**, making LCRQ + Aggregating Funnels the fastest published MPMC queue as of 2025.

## The problem it solves

[[faa-vs-cas|FAA beats CAS]] decisively under contention — but FAA itself is not free. A `LOCK XADD` on a single contended cache line still serializes through the cache coherence protocol, and at 100+ threads even FAA becomes a bottleneck. The hardware physically cannot grant atomic increments faster than one cache-coherence round-trip.

Aggregating Funnels addresses this by *combining* FAA calls in software before they hit the hardware. A funnel is a tree of nodes; threads contend at the leaves, partial increments are merged on the way up, and a single FAA is performed at the root for the combined increment. Each thread then retrieves its individual slot index from the combined result.

## Why it works

The trick is that FAA is *associative*: incrementing by 1 a hundred times produces the same shared-index result as incrementing by 100 once. A funnel exploits this to amortize the cache-coherence cost of the contended atomic across many participants. Latency for an individual operation goes up slightly, but throughput goes up dramatically — and at the thread counts where the funnel is profitable, the per-thread latency reduction from avoided contention exceeds the funnel overhead anyway.

This is the same combining-tree idea that goes back to Yew, Tzeng, and Lawrie's NYU Ultracomputer work (1987), but tuned for modern cache hierarchies rather than network-on-chip routing.

## Performance

On the standard MPMC queue benchmarks at 144 threads on a 4×18-core Intel Xeon:

- **Up to 2.5× higher throughput** than vanilla LCRQ
- Improvement scales with thread count — minimal at 16 threads, peak at 100+
- Negligible overhead at low thread counts where contention is already low

This makes LCRQ + Aggregating Funnels the new ceiling for strict-FIFO MPMC, supplanting plain LCRQ for the first time in 12 years.

## Where it goes from here

Aggregating Funnels is a *layer*, not a queue — the same combining technique applies to any contended FAA. The PPoPP 2025 paper demonstrates it on counters and stacks as well. The natural next step is plugging it into [[lprq|LPRQ]] for portable architectures, and into [[wcq|wCQ]] to give a wait-free queue with the same scalability ceiling.

## See also

- [[lcrq]] — the queue this layer was originally designed for
- [[the-fastest-queue]] — full hierarchy
- [[faa-vs-cas]] — why FAA was the right primitive to combine
