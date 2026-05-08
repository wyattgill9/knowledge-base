---
tags:
  - performance
  - architecture
  - concurrency
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
  - "Raw/Fastest CS/General.md"
last_updated: 2026-05-07
---

# Mechanical Sympathy

Mechanical sympathy is Martin Thompson's term — borrowed from Jackie Stewart's racing philosophy — for designing software that works *with* the hardware rather than against it. The thesis: at the limits of performance, **language choice matters less than understanding the machine**. Java volatile ping-pong runs at 50 ns; C++ atomic ping-pong runs at 45 ns on the same hardware. The 10% gap is rounding error compared to the 100× cost of a single algorithm choice that fights the cache coherence protocol. The [[lmax-disruptor|LMAX Disruptor]] is the canonical demonstration: a Java program that beats most C/C++ queues by aligning every design decision with the underlying hardware reality.

## The core claim

The gap between languages, at the inter-thread communication layer, is far smaller than developers expect:

| Implementation | Latency per op |
|----------------|----------------|
| Java volatile (ping-pong) | 50 ns |
| C++ `std::atomic` (ping-pong) | 45 ns |
| Rust SPSC ring buffer | comparable to C++ |

All three emit `LOCK`-prefixed x86 instructions and bottleneck on the [[cache-coherency|MESI/MOESI protocol]]. The compiler's job is to emit the right instructions; once it does, the hardware decides the speed. Languages diverge meaningfully only at higher levels — GC pauses, allocation patterns, FFI overhead — which can usually be engineered around.

## What "designing with the hardware" actually means

The Disruptor's design choices are the canonical checklist:

- **[[ring-buffer|Pre-allocated ring buffer]]** — no allocator on the hot path, no GC pressure, guaranteed cache-resident data.
- **Power-of-two sizing** — index modulo becomes bitwise AND, one cycle instead of ten.
- **[[false-sharing|Cache-line padding]]** — sequence counters at 64-byte (now 128-byte) alignment so producer and consumer don't fight over the same line.
- **Single-writer principle** — each sequence number is owned by exactly one thread; no atomic RMW needed (see [[spsc-queue]]).
- **Sequence-number coordination** — readers see committed data via monotonic counters rather than locks or CAS.
- **Batch processing** — consumers wake and drain everything available, amortizing wakeup cost.

Each of these is a hardware-aware decision. Together they pull a Java program down to **52 ns per hop** on 2011-era hardware — a number that approaches the bare cache-coherence floor of that machine.

## The corollary: hardware sets the ceiling

Mechanical sympathy implies an inverse principle: **no amount of clever software can go faster than the hardware allows**. The cache-coherence floor on AMD Zen 3 is 16 ns; no algorithm can transfer a cache line between cores faster than that. The L3 hit latency is ~40 cycles; nothing reads from L3 faster. DRAM round-trip is ~80 ns; nothing reads from RAM faster.

This is why optimization at the limits looks like:

1. Identify the hardware operation that bounds the problem.
2. Reduce the number of those operations per logical operation.
3. Accept the per-operation cost; it isn't going down.

Step 2 is where mechanical sympathy lives. The cached-indices optimization in modern [[spsc-queue|SPSC]] queues converts ~3 cache-coherence transactions per push to ~1 per N pushes, a 20× speedup. The [[faa-vs-cas|FAA-over-CAS]] insight converts O(log N) coherence round-trips per increment under contention to O(1). [[flat-combining]] amortizes a single lock acquisition across N pending operations.

## When mechanical sympathy fails

The frame is wrong for two kinds of work:

- **Throughput-by-parallelism**: if you can shard data so threads never share state, mechanical sympathy is irrelevant — there's no inter-thread coordination to optimize. [[thread-per-core]] takes this stance: rather than make shared-state communication fast, eliminate it.
- **Algorithmic dominance**: when one algorithm is asymptotically better, mechanical sympathy of the slower algorithm is a rounding error. A perfectly-tuned O(N²) sort on 10M items still loses to a naive O(N log N).

The frame is right when the algorithm is fixed (you're already using the right queue, the right hash table, the right sort) and the question is how close to the hardware floor you can get.

## What this means in practice

For an engineer:

- **Measure on the target hardware.** Apple M1 vs Intel Sapphire Rapids vs AWS Graviton 3 produce dramatically different absolute numbers; an optimization that wins on one may lose on another. See [[memory-ordering]].
- **Read disassembly when it matters.** The line between "I think this is atomic" and "the compiler emits a `LOCK` prefix" is one `objdump`. Most performance surprises are visible at the instruction level.
- **Cache-line-align contended state** ([[false-sharing|128 bytes on modern x86]]) and **don't share what doesn't need sharing**.
- **Don't fight the prefetcher**: predictable access patterns are almost always faster than clever ones.
- **The language matters last.** A Rust program that copies a 10 MB buffer through a queue is slower than a Java program that hands off a reference. Use the language that lets you express the right hardware-aware design.

## See also

- [[lmax-disruptor]] — the canonical demonstration
- [[cache-coherency]] — the protocol underlying everything
- [[false-sharing]] — the most common mechanical-sympathy violation
- [[inter-thread-communication]] — where the principle gets cashed out
- [[mimalloc]] / [[jemalloc]] — allocator choice as mechanical sympathy
