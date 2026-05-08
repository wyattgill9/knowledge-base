---
tags:
  - rust
  - concurrency
  - crate
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# crossbeam-epoch

Epoch-based memory reclamation for lock-free data structures in Rust. Provides the Rust ecosystem's equivalent of [[rcu|Read-Copy-Update]] — allowing readers to access shared data without locks while writers safely defer deallocation of old versions.

crossbeam-epoch underpins most of the Rust concurrent data structure ecosystem, with crossbeam-deque alone having 299M+ downloads. [[papaya]], [[scc]], and other lock-free Rust structures build on epoch-based or similar reclamation schemes.

## EBR's place in the reclamation landscape

EBR is one of three production-grade reclamation strategies for lock-free structures, and the right choice depends on which trade-off hurts least:

| Scheme | Common-path cost | Worst-case retention | Stalled-thread tolerance |
|--------|------------------|----------------------|--------------------------|
| **EBR** (crossbeam-epoch, folly) | ~zero | Unbounded | Poor — stalled epoch retains memory |
| [[hazard-pointers]] (C++26 standard) | One store + fence per access | Bounded | Good |
| [[crystalline-reclamation\|Crystalline]] (PLDI 2024) | Low | Bounded | Wait-free |

EBR is the right default for general-purpose Rust concurrent data structures because the common-path cost is the lowest of any safe scheme. It is the wrong choice when threads can stall arbitrarily (kernel-adjacent code, hard-real-time systems) — there hazard pointers' bounded retention or Crystalline's wait-freedom matters more than EBR's amortized speed.

For lock-free linked lists specifically — see [[fastest-linked-lists]] — reclamation cost is **the dominant overhead**, often beating the algorithmic cost of the structure itself. The [[bw-tree]]'s loss to [[art-olc|ART-OLC]] is a closely related result: the cost of reclamation traffic on shared cache lines drowns the algorithmic win.
