---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Inter-Thread Communication

The fastest practical inter-thread communication is a busy-spinning [[spsc-queue|SPSC ring buffer]] with cached indices, pinned to cores sharing an L3 cache, achieving **50–135 ns round-trip latency**. The absolute physical floor is the cache-line transfer cost between cores: **16–25 ns** on the best modern hardware ([[cache-coherency|AMD Zen 3 same-CCX or Intel ring bus]]). Every other inter-thread mechanism — atomics, futexes, condition variables, sockets, eventfd — is a layer of additional latency stacked on top of this floor. The engineering challenge is staying as close to the floor as possible while passing meaningful data.

## The hierarchy

The full latency ladder spans roughly six orders of magnitude. Three insights matter more than any single number:

1. **[[core-pinning|Topology trumps algorithm]].** The same queue design varies 5× in latency depending on whether the threads share an L3 (16 ns transfer) or cross an interconnect fabric (107 ns on AMD Infinity Fabric, 150 ns cross-socket on Intel). `taskset` and `isolcpus` are the single highest-leverage optimizations.
2. **[[faa-vs-cas|FAA and CAS cost the same per instruction]]** (~6–7 ns uncontended). The performance gap is semantic, not hardware: FAA always succeeds, while CAS retry storms collapse throughput under contention.
3. **`std::mutex` is not the bottleneck most developers think.** Modern futex-based implementations complete >80% of operations entirely in userspace, adding only ~25 ns over raw atomics. See [[synchronization-primitives]].

## The physical floor

| CPU | Same-cluster | Cross-cluster |
|-----|--------------|---------------|
| AMD Ryzen 9 5900X (Zen 3) | **16 ns** | 84 ns (cross-CCX via Infinity Fabric) |
| Intel i9-9900K (Coffee Lake) | **21 ns** | — (uniform 4-core ring bus) |
| Intel i9-12900K (Alder Lake P→P) | 35 ns | 50 ns (E→E) |
| Intel Xeon Max 9480 (Sapphire Rapids) | 61.5 ns | 150 ns (cross-socket) |
| Apple M1 Pro (P→P) | 40 ns | 145 ns (P→E cluster) |
| AWS Graviton 3 (64 cores) | 46 ns | — (uniform mesh) |

AMD's Zen 3 has the industry-best 16 ns within a CCX (8 cores sharing L3) but balloons to 84–107 ns crossing chiplets. Intel's 4-core consumer parts offer predictable ~21 ns across all cores via a ring bus; server parts with mesh interconnects sit at 48–62 ns. Hyperthreads on the same core can signal each other in **16.5 ns** since they share L1/L2 — but this is rarely practical for data-passing workloads. See [[cache-coherency]] for the underlying [[memory-ordering|MESI/MOESI protocol]] details.

## The synchronization cost ladder

| Level | Mechanism | Typical latency | CPU cost |
|-------|-----------|-----------------|----------|
| 0: Hardware floor | Cache-line transfer | 16–65 ns | Negligible |
| 1: Atomic spin | [[busy-spin\|Busy-wait on atomic flag]] | 35–100 ns RTT | 100% of one core |
| 2: Userspace mutex | Futex fast path (uncontended) | 25–50 ns | Minimal |
| 3: Syscall | Futex slow path / eventfd | 1–5 µs | Low |
| 4: Context switch | Condition variable wake | 1.2–10 µs | Moderate |
| 5: Scheduler | Thread sleep/wake cycle | 5–50 µs | High (cache pollution) |

Surprising data point: **`eventfd` (4,353 ns RTT) is slower than UNIX domain sockets (1,439 ns)** for thread signaling. Both are orders of magnitude slower than atomic-based approaches because every notification is a syscall. See [[synchronization-primitives]] for the full breakdown.

## Why SPSC ring buffers win

Lamport (1977) proved correctness of an SPSC ring buffer that needs **zero atomic read-modify-write operations**. Each index is written by exactly one thread, so plain loads and stores with acquire-release ordering suffice. On x86 with Total Store Order, these compile to ordinary `MOV` instructions — the cheapest possible memory operations.

Three modern optimizations on top of Lamport's design:

- **Local index caching** (Erik Rigtorp): the producer keeps a cached copy of the consumer's read index and only re-fetches when the cache says the queue appears full. Yields **20× throughput improvement** on AMD Ryzen 9 3900X (5.5M → 112M ops/sec) by reducing coherence traffic from ~3 cache misses per operation to near zero amortized.
- **Power-of-two sizing** replaces modulo with bitwise AND.
- **[[huge-pages|Huge-page backing]]** eliminates TLB misses across the ring buffer.

Measured SPSC RTT latencies:

| Implementation | RTT | Throughput | Hardware |
|----------------|-----|------------|----------|
| [[lmax-disruptor\|LMAX Disruptor]] (Java, 1P-1C) | **52 ns/hop** | 25M+ msgs/sec | Intel i7-2720QM |
| [[atomic-queue\|atomic_queue]] SPSC mode | <100 ns | — | various x86 |
| Rigtorp SPSCQueue (C++) | 133 ns | 363K ops/ms | AMD Ryzen 9 3900X cross-chiplet |
| folly::ProducerConsumerQueue | 147 ns | 149K ops/ms | same Ryzen |
| boost::lockfree::spsc | 222 ns | 210K ops/ms | same Ryzen |
| [[rtrb]] (Rust) | comparable to C++ | — | wait-free, ownership-enforced |

The Disruptor's 52 ns per hop on 2011-era hardware is remarkable — approaching the raw core-to-core latency of that hardware. Its secret is the [[lmax-disruptor|single-writer principle]]: each sequence counter is owned by one thread, and consumers batch-process available events when they wake.

## MPMC pays for coordination

Multi-producer multi-consumer queues fundamentally cannot avoid atomic RMW on shared head/tail indices. The progression:

- **[[michael-scott-queue|Michael & Scott (1996)]]** — CAS on linked-list pointers; basis for `ConcurrentLinkedQueue`.
- **Vyukov bounded MPMC (~2010)** — per-slot sequence number with CAS on shared indices.
- **[[lcrq|Morrison & Afek LCRQ (PPoPP 2013)]]** — replaces CAS with FAA on the fast path; current state of the art for x86.
- **[[lprq|LPRQ (PPoPP 2023)]]** — drops LCRQ's double-width CAS, making it portable.
- **[[wcq|Wait-free CQ (SPAA 2022)]]** — fast-path-slow-path methodology.
- **[[aggregating-funnels|Aggregating Funnels (PPoPP 2025)]]** — software FAA combining; up to 2.5× over plain LCRQ.

ETH Zurich (Schweizer, Besta, Hoefler) proved that CAS, FAA, and swap have essentially identical instruction-level latency — the popular belief that "FAA is faster than CAS" comes from FAA's semantic advantage (no wasted retries), not hardware cost. See [[faa-vs-cas]].

Advanced techniques that take a different approach: **[[flat-combining]]** (Hendler et al., 2010) serializes operations through a single combiner thread, outperforming lock-free queues under very high contention. **Elimination** ([[elimination-backoff-stack|Hendler-Shavit-Yerushalmi]]) lets matching push/pop pairs exchange values directly in a side array, bypassing the central structure.

## How HFT firms squeeze out every nanosecond

The high-frequency trading playbook combines several layers, documented at [[core-pinning]]:

- `isolcpus` + `taskset` reduces tail latency from **120 µs to 30 µs** at the 99.9th percentile.
- **1 GB [[huge-pages|huge pages]]** cut TLB misses by ~99% across shared ring buffers.
- **`SCHED_FIFO` real-time priority** prevents preemption.
- **Disabled hyperthreading** eliminates resource contention on execution units.

Intel's 2024 optimization study on Sapphire Rapids hit **450 million 64-byte messages per second** using lazy flow control — updating the read pointer only every Nth message. The `CLDEMOTE` instruction (hints the hardware to push a cache line toward L3 proactively) boosted core-to-core bandwidth from **7.3 GB/sec to 22 GB/sec** when 8+ threads are involved.

[[kernel-bypass|DPDK-style kernel bypass]] applies the same principles to network I/O: poll-mode drivers in tight userspace loops achieve **~5 µs** average UDP latency versus **~9 µs** through the kernel stack. The pattern translates directly to inter-thread work.

## Cross-language convergence

The performance gap between languages for inter-thread communication is far smaller than developers expect. Martin Thompson's canonical benchmark showed Java volatile ping-pong at **50 ns** versus C++ atomic at **45 ns** on identical hardware — only **~10% difference**. Both emit `LOCK`-prefixed x86 instructions that bottleneck on the same cache coherence protocol. [[rtrb|Rust SPSC ring buffers]] produce essentially identical machine code to C++ equivalents. This is the [[mechanical-sympathy|mechanical sympathy]] thesis: the hardware sets the speed, and the language barely matters.

Where languages diverge is in tail latency and GC. Java's GC pauses can inject multi-millisecond stalls into otherwise sub-microsecond paths — the [[lmax-disruptor|Disruptor]] mitigated this through pre-allocated ring entries, and JCTools' SPSC queues use `Unsafe` to match native performance. Go's goroutine channels are convenient but significantly slower than dedicated lock-free queues. For Rust, [[kanal]] leads channel throughput, while [[rtrb|raw SPSC ring buffers]] remain faster for 1P/1C.

## The recipe

The fastest known inter-thread communication converges on:

1. **[[spsc-queue|SPSC ring buffer]]** (Lamport 1977 + cached indices + [[false-sharing|128-byte padding]]).
2. **[[busy-spin|Busy-spin waiting]]** (no syscalls).
3. Threads **[[core-pinning|pinned to cores sharing an L3]]**.
4. **[[huge-pages|Huge pages]]** backing the buffer.

This achieves 50–135 ns RTT — within 2–5× of the bare cache coherence floor. The [[lmax-disruptor]] proved this approach extends to multi-consumer pipelines at 52 ns per hop in Java.

## See also

- [[the-fastest-queue]] — broader queue performance hierarchy
- [[concurrent-queues]] — full MPMC landscape
- [[mechanical-sympathy]] — the design philosophy
- [[thread-per-core]] — server architecture built on these primitives
