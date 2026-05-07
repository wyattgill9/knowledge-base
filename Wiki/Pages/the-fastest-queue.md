---
tags:
  - performance
  - concurrency
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# The Fastest Queue in All of Computer Science

A benchmarked survey of FIFO queue performance in 2025, drawing on every relevant paper from PPoPP, DISC, and SPAA between 2013 and 2025. The hierarchy is remarkably stable: a [[ring-buffer|bounded SPSC ring buffer]] sits at the top with **300–530 million ops/s**, the [[lcrq|LCRQ]] family — augmented by [[aggregating-funnels|Aggregating Funnels]] (PPoPP 2025) — leads strict-FIFO MPMC, and the choice below that depends on what portability, wait-freedom, or NUMA properties you need.

## The single most important insight

[[faa-vs-cas|Fetch-and-add beats compare-and-swap]] at contention hotspots, and it is not close. CAS retries under contention — a phenomenon called *contention meltdown* where throughput *decreases* as thread count rises — while FAA always succeeds in hardware. At 144 threads on x86, FAA completes in roughly 40 ns versus 75 ns for CAS, a 1.8× advantage that compounds across millions of operations. Every fast modern MPMC queue ([[lcrq|LCRQ]], [[scq|SCQ]], [[lprq|LPRQ]], [[wcq|wCQ]]) takes a single FAA on a shared index to claim a slot, then handles slot-level synchronization separately.

The runner-up insight is the [[false-sharing|128-byte cache-line rule]]: modern x86 ships an *adjacent cache-line prefetcher* that speculatively loads the neighbor of any line you touch, so the canonical "pad to 64 bytes" advice is wrong. Padding head and tail indices to 128 bytes apart delivers up to **3.1× over unpadded queues**.

## The performance hierarchy

| Tier | Design | Throughput | When to use |
|------|--------|------------|-------------|
| Tier 0 | Single-threaded [[ring-buffer]] | 300–530M ops/s | Single thread, no synchronization |
| Tier 1 | [[spsc-queue\|SPSC]] ring buffer (acquire/release, shadow indices) | 200–530M ops/s | One producer, one consumer |
| Tier 2 | [[lcrq\|LCRQ]] + [[aggregating-funnels]] (PPoPP 2025) | Highest published MPMC | x86, strict FIFO, max throughput |
| Tier 2 | [[scq\|SCQ]] / [[lprq\|LPRQ]] | ≈LCRQ on x86, wins on PowerPC | Portable MPMC, no CAS2 |
| Tier 3 | [[lmax-disruptor]] | 26M ops/s, 52 ns mean latency | Bounded pipelines, predictable latency |
| Tier 3 | [[moodycamel-concurrent-queue]] | ~114M ops/s 1P/1C | C++ MPMC where strict FIFO not required |
| Tier 4 | [[michael-scott-queue]] | ~10M ops/s | Reference implementation, beaten by everything |

Each tier corresponds to roughly an order of magnitude in throughput. The tier *boundary* is not the algorithm — it is the synchronization style: no atomics → release/acquire → FAA → CAS retry loop.

## Single-threaded: the ring buffer is essentially perfect

A fixed-size [[ring-buffer|circular buffer]] is **roughly 1.5× faster** than C++ `std::queue` (backed by `std::deque`) and **2–10× faster** than linked-list queues once the working set exceeds L1. Index computation is bitwise AND when capacity is a power of two; the array is contiguous so the prefetcher works for free; there is zero allocation on the hot path.

The unrolled (chunked) linked list — multiple elements per node, ideally one cache line's worth — recovers most of the cache performance of arrays with up to **60% fewer cache misses** than standard linked lists, while supporting unbounded growth. C++'s `std::deque` is essentially a less-optimized version of this idea. A virtual-memory trick — mapping the same physical pages twice contiguously — eliminates the wrap-around branch entirely.

## Concurrent: the LCRQ family rules

The [[lcrq|LCRQ]] (Morrison & Afek, PPoPP 2013) outperformed the classic [[michael-scott-queue]] by **>3×** and flat-combining queues by **>2.5×** on 4×18-core Xeons. Its core trick: a single FAA claims a slot, then a 128-bit CAS (`CMPXCHG16B`) reconciles the data exchange. The dependence on CAS2 limited it to x86 for nearly a decade.

Three successors closed the gaps:

- **[[scq|SCQ]]** (Nikolaev, DISC 2019) — single-width CAS plus FAA, matches LCRQ on x86 and **wins on PowerPC**, uses **half the memory** by eliminating per-slot padding. Its threshold mechanism prevents livelock that plagues simpler ring designs.
- **[[lprq|LPRQ]]** (Romanov & Koval, PPoPP 2023) — directly modifies LCRQ to remove all CAS2 while matching its performance, and outruns previous portable designs by 1.6×. Already ported to Java, Kotlin, and Go.
- **[[wcq|wCQ]]** (Nikolaev & Ravindran, SPAA 2022) — wait-free MPMC performing **on par with the best lock-free designs** via fast-path-slow-path: SCQ runs on the fast path; only after MAX_PATIENCE failures does a thread cooperate via the slow path. The slow path almost never fires.

[[aggregating-funnels|Aggregating Funnels]] (PPoPP 2025) is the latest layer: it batches FAA operations across threads in software, eliminating the FAA hotspot itself. Plugged into LCRQ, this gives **up to 2.5× higher throughput at high thread counts**, making LCRQ + Aggregating Funnels the current ceiling for strict-FIFO MPMC.

## SPSC is in a privileged tier

[[spsc-queue|Single-producer single-consumer]] queues require **no atomic read-modify-write instructions** — only acquire loads and release stores, the cheapest possible atomics. Combined with shadow variables (local copies of the remote index, checked before any cross-core read), this converts per-operation cache-line transfers into per-batch transfers.

| Implementation | Throughput | Hardware |
|----------------|------------|----------|
| Naïve SPSC ring buffer (seq_cst) | 5.5M ops/s | AMD Ryzen 9 3900X |
| Optimized SPSC (acquire/release, cached indices) | 112M ops/s | AMD Ryzen 9 3900X |
| Fully optimized C++ SPSC | **305M ops/s** | Intel Core Ultra 5 135U |
| Rust [[rtrb|ringbuffer-spsc]] | **520–530M ops/s** | Apple M4 |

The 20× gap between naïve and optimized on identical hardware comes almost entirely from cache coherence reduction.

## Specialized contexts

**[[numa-aware-queues|NUMA]]** — On multi-socket systems, NUMA Node Delegation (Nuddle) routes operations through per-node server threads to keep cache lines on the local interconnect, achieving 1.87× over NUMA-oblivious designs. The Linux kernel's NUMA-aware qspinlock applies the same idea to the lock case.

**[[gpu-queues|GPU]]** — 160,000+ concurrent threads, no coherent L1, SIMT execution. CAS-based queues melt down catastrophically (one benchmark: 1,112× slower than a blocking array queue at high thread counts). Warp-centric designs aggregating ops within a 32-thread warp give up to 40× over thread-centric. [[bacq|BACQ]] (2024) is the current state of the art.

**Bounded vs unbounded** — Bounded ring buffers achieve O(1) worst-case with zero allocation and natural backpressure. Unbounded queues (chunked linked lists) introduce 20–100+ ns latency spikes on segment allocation and create GC pressure in managed languages. For HFT, audio DSP, motor control: bounded only.

**[[persistent-queues|Persistent (functional)]]** — Hood-Melville real-time queues (1981), Okasaki's scheduling technique (1995/1998), and the Kaplan-Tarjan deque with O(1) catenation (1999) all achieve worst-case O(1). The Kaplan-Tarjan deque is so complex that a verified OCaml implementation only landed in 2025.

## Per-language fastest

| Language | Fastest library | 1P/1C throughput |
|----------|-----------------|------------------|
| C++ | [[atomic-queue]] (OptimistAtomicQueue) | 200–500M+ ops/s |
| Rust | [[rtrb]] / [[crossbeam-array-queue\|crossbeam ArrayQueue]] | ~100–300M ops/s |
| Java | JCTools `MpmcArrayQueue` | ~65M ops/s |
| Java (framework) | [[lmax-disruptor]] | ~26M ops/s, 52 ns latency |
| Go | Lock-free ring buffer | ~5–20M ops/s |
| C | Concurrency Kit (`ck`) | Comparable to C++ |

C++ queues run **2–3× faster** than Java's best for raw throughput. Go channels are **10–50× slower** than optimized C++; lock-free Go ring buffers narrow the gap to 3–5×. Python is 100–1,000× slower than native — `faster-fifo` (C++-backed IPC) closes ~30× of the gap.

## What changed and what didn't

The hierarchy of *primitives* — FAA over CAS, ring buffer over linked list, padding over none — has been stable since 2013. What has changed is that the missing pieces (portability, wait-freedom, scalability past hardware FAA throughput) have been filled in:

- **Portability** — closed by SCQ (2019) and LPRQ (2023).
- **Wait-freedom** — closed by wCQ (2022): the fast-path-slow-path methodology demonstrated that wait-freedom no longer costs throughput.
- **FAA scalability** — addressed by Aggregating Funnels (2025): software-combined FAA defeats the hardware bottleneck.

The 2025 picture is that LCRQ + Aggregating Funnels owns strict-FIFO MPMC, an optimized SPSC ring buffer dominates one-producer-one-consumer, and the [[lmax-disruptor]] remains the latency king for bounded pipelines. The remaining open problem — the November 2025 preprint **Cyclic Memory Protection** (CMP) — claims coordination-free memory reclamation at 6.49M items/s, but is not yet peer-reviewed.
