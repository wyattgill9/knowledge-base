---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Memory Ordering

Memory ordering is the contract between a CPU and the programmer about how operations on different memory locations may be reordered. **x86 has Total Store Order (TSO)** — store-store and load-load ordering are guaranteed natively, so most acquire-release atomics compile to plain `MOV` instructions with zero overhead. **ARM has a relaxed model** — explicit `DMB` (Data Memory Barrier) instructions are required, costing 10–20 ns each. This is why many lock-free patterns measurably run faster on x86 than ARM, even on otherwise comparable hardware.

## The hardware models

| Architecture | Model | Default ordering | Cost of explicit barrier |
|--------------|-------|------------------|--------------------------|
| x86 / x86-64 | TSO | Loads ordered, stores ordered, only store→load may reorder | `MFENCE` ~30–60 ns |
| ARM (ARMv8) | Relaxed (with one-way fences) | Almost any reorder allowed | `DMB ISH` ~10–20 ns |
| POWER | Weakly ordered | Almost any reorder allowed | `lwsync` / `sync` ~15–30 ns |
| RISC-V | Relaxed (with FENCE) | Configurable per fence | `FENCE rw,rw` ~10–20 ns |

x86's TSO is the most permissive model in common use: among the four reorder pairs (load-load, load-store, store-load, store-store), only store→load can be reordered. Almost every ordering programmers want "for free" is already provided. ARM's relaxed model leaves all four pairs free to reorder unless an explicit barrier prevents it.

## What this means for atomics

C++/Rust memory orderings map to instructions like this on x86:

| Ordering | x86 emission |
|----------|--------------|
| `relaxed` load/store | plain `MOV` |
| `acquire` load | plain `MOV` (TSO gives load-acquire for free) |
| `release` store | plain `MOV` (TSO gives store-release for free) |
| `acq_rel` RMW | `LOCK`-prefixed RMW |
| `seq_cst` store | `MOV` + `MFENCE`, or `XCHG` |

The takeaway: **acquire-release synchronization is free on x86**. SPSC queues that use `Ordering::Acquire` for loads and `Ordering::Release` for stores compile to ordinary MOVs and run at full memory-bandwidth speed. Only `seq_cst` stores require a fence, costing 30–60 ns of pipeline stall.

On ARM, the same code emits explicit `LDAR` (load-acquire) and `STLR` (store-release) instructions which are slightly more expensive than plain loads/stores, plus `DMB` barriers around `seq_cst`. The cost is small per op but compounds in tight loops.

## The seq_cst trap

`seq_cst` (sequential consistency) is the default `std::atomic<T>` ordering in C++ and the most intuitive — every thread sees the same global order of all atomic operations. It's also the most expensive: every store requires a full memory fence, draining the store buffer.

In hot loops, this difference is dramatic. A naive [[spsc-queue|SPSC ring buffer]] using seq_cst hits **5.5M ops/s** on AMD Ryzen 9 3900X. The same buffer with acquire-release ordering hits **112M ops/s** — a **20× improvement** purely from dropping unnecessary fences. (The full optimized version with cached indices also stays at acquire-release.)

The rule: **use `relaxed` for counters that don't synchronize anything, `acquire`/`release` for synchronization, and `seq_cst` only when you genuinely need a global total order across multiple atomics**. Most code that uses `seq_cst` doesn't actually need it.

## Why ARM patterns sometimes need rework

A C++ lock-free algorithm that's correct under x86 TSO can be incorrect under ARM's relaxed model if the author relied implicitly on store ordering. The classic example: a publish-subscribe pattern where the producer writes data to a slot, then sets a "ready" flag, and the consumer reads the flag, then reads the data. On x86, this works with plain stores because store→store ordering is guaranteed. On ARM, the consumer can observe the "ready" flag set *before* the data is visible, leading to a torn read.

The fix is explicit acquire-release ordering on the flag, which compiles to a real barrier on ARM and is free on x86. Code written portably from the start (with `Ordering::Release` on the producer flag write, `Ordering::Acquire` on the consumer flag read) works on both architectures with no penalty on x86.

## PAUSE and YIELD: per-architecture spin hints

The instruction telling the CPU "I'm in a spin loop, slow down" varies:

- **x86**: `PAUSE` — see [[busy-spin]] for the **15× variance** between Broadwell (~9 cycles) and Skylake (~140 cycles).
- **ARM**: `YIELD` — much cheaper than x86 `PAUSE`, typically a few cycles, because ARM doesn't have the same store-buffer flushing cost.
- **Apple Silicon**: `WFE` (Wait For Event) is the recommended spin idiom, paired with `SEV` from the unblocker. Significantly more efficient than `YIELD` for long waits.

This is one place where portable spin loops are genuinely hard to write well. The `std::hint::spin_loop()` in Rust compiles to the right instruction per target, but the cycle cost varies enough that the *number* of spins between checks should also be tuned per platform.

## See also

- [[cache-coherency]] — the underlying protocol
- [[busy-spin]] — where memory ordering meets the PAUSE instruction
- [[faa-vs-cas]] — the same instructions cost the same regardless of ordering
- [[spsc-queue]] — where the seq_cst → acquire-release win is largest
- [[mechanical-sympathy]] — knowing the hardware as a precondition
