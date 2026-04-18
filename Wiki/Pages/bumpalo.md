---
tags:
  - rust
  - performance
  - data-structures
  - crate
sources:
  - "Raw/Rust/The blazingly fast Rust crate stack for 2025–2026.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# bumpalo

bumpalo provides bump allocation in Rust — ~2 ns per allocation with instant bulk deallocation. It's ideal for **phase-oriented work** where you allocate many objects, process them, then discard everything at once (parse -> process -> discard).

## How it works

A bump allocator maintains a single pointer into a pre-allocated region. Each allocation just advances the pointer — no free lists, no metadata per allocation. Deallocation happens all at once when the arena is dropped or reset. This makes individual allocations nearly free but means you can't free individual objects.

## Related arena crates

- **[[slotmap]]** — generational-index arenas that prevent ABA problems, with three variants optimized for different access patterns. Better when you need stable handles and individual removal.
- **[[bytes]]** — `Arc`-backed `Bytes`/`BytesMut` for zero-copy buffer sharing throughout the [[tokio]] ecosystem. Not an arena, but solves a related problem (avoiding copies).

Arena/bump allocators deliver **2–5x speedup** over general malloc for batch allocations (LLVM's `BumpPtrAllocator`, Rust's bumpalo). Combined with [[mimalloc]] as the global allocator, this covers the full allocation performance spectrum. See [[soa-vs-aos]] and [[ecs-pattern]] for how memory layout optimization compounds with allocation optimization.
