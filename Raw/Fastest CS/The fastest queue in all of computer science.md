**The FAA-based ring buffer dominates concurrent queuing, and a plain circular buffer remains unbeatable for single-threaded work.** LCRQ (Linked Concurrent Ring Queue), enhanced with Aggregating Funnels in 2025, holds the crown for multi-producer multi-consumer throughput — delivering up to **2.5× higher throughput** than the previous best at high thread counts. For single-threaded or single-producer single-consumer scenarios, an optimized ring buffer achieves **300–530 million operations per second** on modern hardware, a figure no other data structure touches. The story of queue performance is fundamentally a story about cache lines, atomic instructions, and the gap between CAS (which can fail) and FAA (which never does).

---

## Single-threaded: the ring buffer is essentially perfect

For single-threaded FIFO queues, the theoretical lower bound is **Ω(1)** per operation, and a fixed-size circular buffer achieves this trivially with the lowest possible constant factors. The array is contiguous in memory, enabling hardware prefetching. Index computation uses bitwise AND (when capacity is a power of two) instead of modulo. There is zero heap allocation during operation. The result is a data structure where enqueue and dequeue each compile to a handful of instructions.

Benchmarks confirm the dominance. A circular buffer is roughly **1.5× faster** than C++'s `std::queue` (backed by `std::deque`) for push/pop workloads, because `std::deque`'s chunked architecture requires two pointer dereferences per access versus one for a ring buffer. Linked-list queues fare far worse — scattered heap nodes defeat the prefetcher, and per-node allocation adds **50–100+ nanoseconds** of overhead per operation. For queues exceeding L1 cache size, linked-list traversal runs **2–10× slower** than contiguous structures.

The main alternative worth considering is the **unrolled (chunked) linked list**, which stores multiple elements per node (typically one cache line's worth). This design recovers most of the cache performance of arrays — up to **60% fewer cache misses** than standard linked lists — while supporting unbounded growth without full-buffer copies. C++'s `std::deque` is essentially a less-optimized version of this idea. For unbounded single-threaded queues, a growable ring buffer (doubling on overflow) provides amortized O(1) with excellent cache behavior, though individual resize operations cost O(n).

A virtual memory trick can eliminate even the branch for wrap-around: map the same physical buffer pages twice contiguously in virtual address space, so reads and writes near the boundary see a seamless continuation. This removes the last conditional from the hot path.

---

## Concurrent queues: FAA-based designs changed everything

The single most important insight in modern concurrent queue design is that **Fetch-And-Add (FAA) beats Compare-And-Swap (CAS)** at contention hotspots. CAS can fail under contention and must retry, wasting CPU cycles — a phenomenon called "contention meltdown" where throughput _decreases_ as thread count rises. FAA always succeeds: the hardware guarantees atomic completion, and x86's `LOCK XADD` is implemented more efficiently in the cache coherence protocol than contested `LOCK CMPXCHG`. At **144 threads**, FAA operations complete in roughly **40 ns** versus **75 ns** for CAS — a 1.8× advantage that compounds across millions of operations.

**LCRQ** (Morrison and Afek, PPoPP 2013) leveraged this insight to create what remains the fastest MPMC queue family. LCRQ is a linked list of fixed-size concurrent ring buffers (CRQs), each typically holding 4,096 entries. Each enqueue or dequeue executes a single FAA on a shared index to claim a slot, then uses a 128-bit CAS (`CMPXCHG16B` on x86) to synchronize the actual data exchange. This design outperforms the classic Michael-Scott lock-free queue by **>3×** and flat-combining queues by **>2.5×** in published benchmarks on 4×18-core Intel Xeon systems.

The newest evolution came in March 2025: **Aggregating Funnels** (Roh, Wei, Ruppert et al., PPoPP 2025) replace LCRQ's hardware FAA hotspots with a software-combining layer that batches FAA operations across threads. Plugged into LCRQ, this eliminates the FAA scalability bottleneck and improves throughput by **up to 2.5×** at high thread counts — making LCRQ + Aggregating Funnels the fastest published MPMC queue as of 2025.

### The portable alternatives: SCQ and LPRQ

LCRQ's dependence on 128-bit CAS (CAS2) limited it to x86. Two successors solved this:

- **SCQ** (Nikolaev, DISC 2019) is a lock-free bounded MPMC ring buffer using only single-width CAS and FAA. It matches LCRQ performance on x86 and **wins outright on PowerPC** (where CAS2 is unavailable). SCQ also uses roughly **half the memory** of LCRQ by eliminating per-slot cache-line padding. Its "threshold" mechanism prevents the livelocks that plague simpler ring-buffer designs.
    
- **LPRQ** (Romanov and Koval, PPoPP 2023) directly modifies LCRQ to remove all CAS2 usage while maintaining identical performance. It **outperforms the best previous non-CAS2 solutions by up to 1.6×** and has been ported to Java, Kotlin, and Go.
    

### Wait-freedom without sacrificing speed

**wCQ** (Nikolaev and Ravindran, SPAA 2022) achieved what was long thought impractical: a wait-free MPMC queue performing **on par with the best lock-free designs**. It uses a fast-path-slow-path methodology — the fast path runs SCQ's lock-free algorithm, and only when a thread fails after MAX_PATIENCE attempts does it switch to a cooperative slow path where all threads assist the stuck operation. In practice, the slow path triggers so rarely that overhead is negligible. This confirms the modern consensus: with careful design, **wait-freedom and high throughput are no longer mutually exclusive**.

### SPSC queues are 5–50× faster than MPMC

Single-producer single-consumer ring buffers occupy a privileged performance tier because they require **no atomic read-modify-write instructions** — only atomic loads (acquire) and stores (release), the cheapest possible atomic operations. Key optimizations include cache-line separation of head/tail indices, local caching of the remote index to batch cross-core transfers, and relaxed memory ordering.

|Implementation|Throughput|Hardware|
|---|---|---|
|Naïve SPSC ring buffer (seq_cst)|5.5M ops/s|AMD Ryzen 9 3900X|
|Optimized SPSC (acquire/release, cached indices)|112M ops/s|AMD Ryzen 9 3900X|
|Fully optimized C++ SPSC|**305M ops/s**|Intel Core Ultra 5 135U|
|Rust ringbuffer-spsc|**520–530M ops/s**|Apple M4|

The 20× gap between naïve and optimized implementations on the same hardware comes almost entirely from reducing cache coherence traffic — caching the remote index locally means cross-core cache-line transitions drop from ~3 per operation to ~1 per batch.

---

## Hardware awareness makes or breaks queue performance

### Cache lines and the 128-byte rule

The most impactful hardware optimization is **preventing false sharing** between a queue's head (consumer) and tail (producer) indices. When both indices share a 64-byte cache line, every producer write invalidates the consumer's cache, and vice versa. The standard fix is 64-byte padding, but modern x86 CPUs have an **adjacent cache-line prefetcher** that speculatively fetches neighboring lines. Using **128-byte isolation** (two cache lines of padding) defeats this prefetcher and delivers a **3.1× performance improvement** over unpadded queues in benchmarks.

Beyond spatial separation, **shadow variables** — local copies of the remote thread's index — reduce temporal sharing. The local copy is checked first; only when it indicates the queue may be full or empty does the thread perform a cross-core read. This converts per-operation cache-line transfers into per-batch transfers.

### NUMA-aware queues use delegation

On multi-socket NUMA systems, remote memory access adds significant latency from interconnect traversal. **NUMA Node Delegation (Nuddle)** transforms NUMA-oblivious data structures into NUMA-aware ones by running dedicated "server" threads per NUMA node that execute operations on behalf of client threads. SmartPQ (Giannoula et al., 2024) achieves **1.87× speedup** over NUMA-oblivious designs using this approach. The Linux kernel's NUMA-aware qspinlock, merged around 2021, similarly splits wait queues by NUMA node, approaching **2× speedup** under heavy contention.

### GPU queues demand warp-centric design

GPU queues face fundamentally different challenges: **160,000+** concurrently active threads, no coherent L1 caches, and SIMT execution requiring warp-level coordination. Warp-centric queue designs aggregate operations within a 32-thread warp before touching shared state, achieving up to **40× speedup** over thread-centric designs. LCRQ ported to NVIDIA K20c GPUs reached **201.8 million ops/s** at 623 threads. CAS-based designs suffer catastrophic contention meltdown on GPUs — one benchmark showed a CAS-based queue running **1,112× slower** than a blocking array queue at high thread counts. The BACQ (Boundary-Aware Concurrent Queue, 2024) achieves **2–9×** improvement over existing GPU queue designs.

---

## Bounded queues win on latency; unbounded queues trade jitter for flexibility

Bounded ring buffers achieve **O(1) worst-case** operations with zero allocation, minimal jitter, and natural backpressure. The **LMAX Disruptor** (Java) exemplifies the bounded approach at its finest: it pre-allocates all event objects in ring buffer slots at startup, eliminating garbage collection entirely from the hot path. The results speak for themselves:

|Metric|ArrayBlockingQueue|LMAX Disruptor|
|---|---|---|
|Throughput (1P-1C)|5.3M ops/s|**26.0M ops/s**|
|Mean latency|32,757 ns|**52 ns**|
|99th percentile latency|2,097,152 ns|**128 ns**|

That is **630× lower mean latency** and **8× higher throughput**. The Disruptor processes **6 million orders per second** on a single JVM thread at LMAX Exchange.

Unbounded queues sacrifice these properties for the ability to grow indefinitely. The best unbounded designs use chunked linked lists (allocating new array segments as needed), which provide O(1) worst-case enqueue and reasonable cache locality within segments. However, occasional memory allocation introduces **latency spikes of 20–100+ ns**, memory fragmentation degrades locality over time, and in managed languages, allocation creates GC pressure. For latency-critical systems (HFT, audio, motor control), bounded queues are strictly preferred.

---

## The fastest queue in every major language

**C++** has the most competitive ecosystem. The `atomic_queue` library (OptimistAtomicQueue) consistently achieves the **highest throughput and lowest latency** across benchmarks — sub-100ns round-trip push-to-pop latency in MPMC mode, with SPSC throughput exceeding 200M ops/s. For general-purpose MPMC use, `moodycamel::ConcurrentQueue` delivers ~**114M ops/s** single-threaded with excellent scaling by using per-producer sub-queues (trading strict cross-producer FIFO ordering for throughput). Facebook's `folly::ProducerConsumerQueue` is among the fastest SPSC options at ~67M ops/s.

**Rust** achieves essentially identical performance to C++ thanks to zero-cost abstractions. The `crossbeam` crate provides the standard lock-free MPMC (`ArrayQueue`) and unbounded (`SegQueue`) implementations, with epoch-based memory reclamation adding minimal overhead. For SPSC, `rtrb` and `bounded-spsc-queue` deliver wait-free operation with ~7ns push/pop latency. No direct LCRQ port exists in Rust yet — the closest equivalents use Vyukov-style bounded ring buffers.

**Java** compensates for managed-language overhead with exceptional libraries. **JCTools** `MpmcArrayQueue` achieves **65M ops/s** in 1P1C mode — roughly 50–65% of equivalent C++ performance. The **LMAX Disruptor** achieves 52ns mean latency through "mechanical sympathy" (cache-line padding, single-writer principle, pre-allocation). Java's standard `ConcurrentLinkedQueue` (Michael-Scott algorithm) manages only ~10.7M ops/s, and `ArrayBlockingQueue` ~5.3M ops/s. **C++ queues are roughly 2–3× faster than Java's best** for raw throughput.

**Go** channels run roughly **10–50× slower** than optimized C++ queues. Lock-free Go implementations narrow the gap to ~3–5×. **Python** is not competitive for high-performance queuing: the GIL, interpreter overhead, and serialization costs for multiprocessing make it **100–1,000× slower** than native implementations, though `faster-fifo` (a C++-backed IPC queue) achieves up to 30× improvement over `multiprocessing.Queue`.

|Language|Fastest Library|Throughput (1P1C)|
|---|---|---|
|C++|atomic_queue (OptimistAtomicQueue)|200–500M+ ops/s|
|Rust|rtrb / crossbeam ArrayQueue|~100–300M ops/s|
|Java|JCTools MpmcArrayQueue|~65M ops/s|
|Java (framework)|LMAX Disruptor|~26M ops/s, 52ns latency|
|Go|Lock-free ring buffer|~5–20M ops/s|
|C|Concurrency Kit (ck)|Comparable to C++|

---

## Recent research confirms FAA-ring-buffer supremacy

The most significant papers from 2020–2026 tell a consistent story: FAA-based ring buffers remain the foundation of the fastest queues, with recent work focused on portability, wait-freedom, and scalability:

- **Aggregating Funnels** (PPoPP 2025) — Currently the fastest MPMC design, improving LCRQ by 2.5× at high thread counts through software combining of FAA operations
- **LPRQ** (PPoPP 2023) — Made LCRQ portable to all architectures without CAS2, matching its performance while outrunning previous portable designs by 1.6×
- **wCQ** (SPAA 2022) — Proved wait-freedom can be "free" in practice via fast-path-slow-path methodology
- **SCQ** (DISC 2019/2020) — Best memory-efficient portable alternative, using half the memory of LCRQ
- **Memory Bounds for Bounded Queues** (PPoPP 2024) — Established theoretical limits on how little memory overhead non-blocking bounded queues require
- **"No Cords Attached"** (arXiv, November 2025) — Introduced Cyclic Memory Protection (CMP), claiming 6.49M items/s at 1P1C with coordination-free reclamation (preprint, not yet peer-reviewed)
- **BACQ** (2024) — Boundary-Aware Concurrent Queue for GPUs, achieving 2–9× over existing GPU queues

---

## Worst-case O(1) is achievable — with trade-offs

For **mutable (imperative) queues**, a fixed-size ring buffer provides O(1) worst-case trivially. For **persistent (functional) queues**, the answer is also yes, through increasingly sophisticated techniques:

The **Hood-Melville real-time queue** (1981) was the first persistent queue with O(1) worst-case operations, using global rebuilding to incrementally reverse the rear list. **Okasaki's real-time queue** (1995/1998) simplified this with a "scheduling" technique: maintaining a list of pending suspended computations and forcing one per operation converts amortized costs to worst-case costs. The most ambitious result is the **Kaplan-Tarjan real-time deque** (1999), which supports push, pop, inject, eject, _and_ catenation (appending two deques) — all in O(1) worst-case. It is so complex that a verified OCaml implementation was only achieved in 2025.

The practical trade-off is clear: worst-case O(1) functional queues have **higher constant factors** than amortized versions due to incremental rebuilding overhead. For hard real-time systems (audio DSP at 48kHz gives ~21μs per sample), this overhead is worth paying. For everything else, amortized O(1) with a ring buffer is preferred.

---

## Conclusion

The hierarchy of queue performance is remarkably stable: **bounded SPSC ring buffers** (300–530M ops/s) sit at the top, followed by **LCRQ-family MPMC queues** enhanced with Aggregating Funnels, then **portable alternatives** like SCQ and LPRQ for non-x86 architectures. The key engineering insights — FAA over CAS, 128-byte cache-line isolation, shadow variables for batched cross-core communication, and pre-allocation to eliminate GC — each contribute multiplicative improvements.

The most surprising finding from recent research is that **wait-freedom is no longer expensive**. wCQ demonstrated that the fast-path-slow-path methodology adds negligible average-case overhead while guaranteeing bounded per-operation completion. The theoretical result that lock-free algorithms are "practically wait-free" (Alistarh, Censor-Hillel, and Shavit) suggests that for most applications, the distinction between lock-free and wait-free matters less than getting the cache behavior and atomic instruction choices right.

No single "fastest queue" exists in isolation — the answer depends on producer/consumer topology, thread count, NUMA topology, architecture, and ordering requirements. But if forced to name one design, **LCRQ with Aggregating Funnels** (PPoPP 2025) represents the current ceiling for strict-FIFO MPMC throughput, while an optimized SPSC ring buffer remains the fastest concurrent queue achievable when the use case permits it.