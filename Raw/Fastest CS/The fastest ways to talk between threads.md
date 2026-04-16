**A busy-spinning SPSC ring buffer with cached indices, pinned to cores sharing an L3 cache, is the fastest practical method for inter-thread data passing — achieving round-trip latencies of 50–135 ns.** The absolute physical floor is set by cache coherence: transferring a single cache line between two cores takes **16–25 ns** on the best modern hardware (AMD Zen 3 same-CCX or 4-core Intel ring bus). Every inter-thread communication method ultimately bottlenecks on this cache-to-cache transfer cost, and the engineering challenge is getting as close to it as possible while passing meaningful data. Understanding why requires a tour through cache coherence protocols, atomic primitives, lock-free algorithms, and three decades of academic research.

---

## Cache coherence sets the speed of light

The fundamental unit of inter-thread communication is a **64-byte cache line** bouncing between cores via the CPU's coherence protocol. On Intel, the MESIF protocol tracks whether each line is Modified, Exclusive, Shared, Invalid, or Forward. On AMD, the MOESI protocol adds an Owned state that avoids expensive write-backs. When Thread A writes data and Thread B reads it, the cache line must transition from Modified in A's L1 to Shared in B's L1 — a round-trip through the L3 cache (or interconnect fabric) that defines the latency floor.

Measured core-to-core transfer latencies reveal dramatic variation by topology:

|CPU|Same-cluster latency|Cross-cluster latency|
|---|---|---|
|AMD Ryzen 9 5900X (Zen 3)|**16 ns**|84 ns|
|Intel i9-9900K (Coffee Lake)|**21 ns**|— (uniform ring bus)|
|Intel i9-12900K (Alder Lake, P→P)|35 ns|50 ns (E→E)|
|Intel Xeon Max 9480 (Sapphire Rapids, same socket)|**61.5 ns**|150 ns (cross-socket)|
|Apple M1 Pro (P→P)|40 ns|145 ns (P→E cluster)|
|AWS Graviton 3 (64 cores)|46 ns|— (uniform)|

AMD's Zen 3 achieves the industry-best **16 ns between physical cores** within a CCX (8 cores sharing L3), but crossing the Infinity Fabric to another chiplet balloons to 84–107 ns. Intel's 4-core consumer parts offer a predictable ~21 ns across all cores via a ring bus, while server parts with mesh interconnects average 48–62 ns. **Hyperthreads on the same core** can signal each other in just **16.5 ns** since they share L1/L2 caches — but this is rarely practical for data-passing workloads.

Two hardware effects dominate everything: **false sharing** and **memory ordering costs**. False sharing — where independent variables share a 64-byte cache line — causes 25–50× slowdowns in concurrent counter benchmarks. Every high-performance queue pads its head and tail indices to separate cache lines using `alignas(64)` (C++), `@Contended` (Java), or `CachePadded<T>` (Rust's crossbeam). Memory ordering on x86 is mostly free thanks to Total Store Order (TSO), which guarantees store-store and load-load ordering natively. ARM's relaxed model requires explicit barriers (`DMB`), costing 10–20 ns each, which is why many lock-free patterns run measurably faster on x86.

---

## Why SPSC ring buffers win

The single-producer single-consumer (SPSC) ring buffer, first proven correct by **Leslie Lamport in 1977**, is the fastest queue topology because it requires **zero atomic read-modify-write operations**. Each index (head, tail) is written by exactly one thread, so plain loads and stores with acquire-release ordering suffice. On x86 with TSO, these compile to ordinary `MOV` instructions — the cheapest possible memory operations.

Modern implementations add three critical optimizations to Lamport's original design. First, **local index caching**: the producer keeps a cached copy of the consumer's read index and only fetches the true value when the cache says the queue appears full. Erik Rigtorp demonstrated this single optimization yields a **20× throughput improvement** on AMD Ryzen 9 3900X — from 5.5M to **112M ops/sec** — by reducing coherence traffic from ~3 cache misses per operation to near zero in the amortized case. Second, **power-of-two sizing** replaces expensive modulo with bitwise AND. Third, **huge page backing** eliminates TLB misses across the ring buffer (a 2 MB huge page covers 32× more address space than standard 4 KB pages).

Measured SPSC queue latencies across implementations:

|Implementation|RTT Latency|Throughput|Notes|
|---|---|---|---|
|Rigtorp SPSCQueue (C++)|**133 ns**|363K ops/ms|AMD Ryzen 9 3900X, cross-chiplet|
|atomic_queue SPSC mode (C++)|**<100 ns**|—|Claims sub-100ns, various x86|
|LMAX Disruptor (Java, 1P-1C)|**52 ns/hop**|25M+ msgs/sec|Intel i7-2720QM, BusySpinWait|
|folly::ProducerConsumerQueue (C++)|147 ns|149K ops/ms|Same Ryzen setup|
|boost::lockfree::spsc (C++)|222 ns|210K ops/ms|Same Ryzen setup|
|rtrb (Rust)|Comparable to C++|—|Wait-free, ownership-enforced SPSC|

The Disruptor's **52 ns per hop** on 2011-era Intel is remarkable — approaching the raw core-to-core latency of that hardware. Its secret is the single-writer principle: each sequence counter is owned by one thread, and consumers batch-process all available events when they wake, amortizing overhead. Martin Thompson's "mechanical sympathy" ping-pong benchmark showed Java volatile writes achieve **50 ns** per operation versus C++'s **45 ns** — only a ~10% gap — because the JIT generates nearly identical `LOCK`-prefixed instructions.

---

## MPMC queues pay for coordination with atomics

Multi-producer multi-consumer queues fundamentally cannot avoid atomic read-modify-write operations on shared head/tail indices. The **Michael-Scott queue** (1996) — the foundational lock-free MPMC design — uses CAS on linked-list pointers and became the basis for Java's `ConcurrentLinkedQueue`. **Vyukov's bounded MPMC queue** (~2010) uses a per-slot sequence number with CAS on shared indices, achieving one CAS per operation with excellent cache behavior from its array layout. The breakthrough came from Morrison and Afek's **LCRQ** (PPoPP 2013), which replaced CAS with **fetch-and-add (FAA)** on the fast path. FAA always succeeds (unlike CAS, which retries under contention), making LCRQ the acknowledged **fastest MPMC queue** for x86 and the current state of the art.

Uncontended atomic operation costs on Intel Haswell: CAS **~6 ns**, FAA **~7 ns**, plain load **~1.2 ns**. Contended (2 threads, Skylake): atomic add **~50 ns/op**, CAS-based add **~65 ns/op**, `std::mutex`-protected add **~125 ns/op**. The ETH Zurich study by Schweizer, Besta, and Hoefler proved that CAS, FAA, and swap have essentially identical instruction-level latency — the popular belief that "FAA is faster than CAS" comes from FAA's semantic advantage (no wasted retries), not hardware cost.

Key academic milestones tell a clear progression:

- **Herlihy (1991)**: Proved CAS is "universal" — it can implement any concurrent object wait-free. Established the consensus hierarchy showing read-write registers cannot solve consensus for 2+ threads.
- **Michael & Scott (1996)**: The definitive lock-free linked-list queue, still the most deployed algorithm.
- **Morrison & Afek (2013)**: LCRQ demonstrated FAA-based queues outperform CAS-based designs under contention.
- **Yang & Mellor-Crummey (2016)**: First wait-free queue matching FAA throughput, using fast-path-slow-path methodology.
- **LPRQ (PPoPP 2023)**: Eliminated LCRQ's double-width CAS requirement, making it portable to Java, Go, and ARM.

Advanced techniques like **flat combining** (Hendler et al., 2010) take a counterintuitive approach: instead of parallelizing data structure access, they serialize it through a single combiner thread that batches operations. This outperforms lock-free queues under very high contention because it eliminates CAS retry storms entirely. **Elimination** (Hendler, Shavit, Yerushalmi) lets matching push/pop operations exchange values directly in a side array, bypassing the central structure.

---

## The synchronization cost ladder from nanoseconds to microseconds

Every inter-thread notification mechanism falls on a latency ladder determined by how deeply it touches the operating system:

|Level|Mechanism|Typical latency|CPU cost|
|---|---|---|---|
|**0: Hardware floor**|Cache-line transfer|16–65 ns|Negligible|
|**1: Atomic spin**|Busy-wait on atomic flag|35–100 ns RTT|100% of one core|
|**2: Userspace mutex**|Futex fast path (uncontended)|25–50 ns|Minimal|
|**3: Syscall**|Futex slow path / eventfd|1–5 µs|Low|
|**4: Context switch**|Condition variable wake|1.2–10 µs|Moderate|
|**5: Scheduler**|Thread sleep/wake cycle|5–50 µs|High (cache pollution)|

The x86 `PAUSE` instruction dramatically changed behavior across generations: **~9 cycles on Broadwell**, then **~140 cycles on Skylake** (a 15× increase Intel made to improve SMT fairness). This means spin-wait loop tuning is architecture-dependent. Modern `pthread_mutex` implementations are adaptive: glibc spins briefly checking if the lock holder is running on another core before falling back to `futex(FUTEX_WAIT)`. Travis Downs measured that under moderate contention (3 threads), only **~0.18 futex syscalls occur per mutex operation** — over 80% complete entirely in userspace.

The `eventfd` mechanism, surprisingly, is slower than UNIX domain sockets for signaling: **4,353 ns** median RTT versus **1,439 ns** for `AF_UNIX`. Both are orders of magnitude slower than atomic-based approaches because every notification requires a system call. `io_uring` reduces this overhead for I/O via shared-memory submission/completion rings, but for pure inter-thread signaling, futex-based approaches remain faster.

---

## How HFT firms squeeze out every nanosecond

High-frequency trading systems represent the extreme end of inter-thread optimization, and their techniques reveal what matters most at the margin. The playbook combines several layers: **core pinning** (`isolcpus` + `taskset`) to eliminate scheduler interference, which alone reduces tail latency from **120 µs to 30 µs** at the 99.9th percentile. **1 GB huge pages** cut TLB misses by ~99% across shared ring buffers. **`SCHED_FIFO` real-time priority** prevents preemption. **Disabled hyperthreading** eliminates resource contention on execution units.

Intel's own 2024 optimization study on Sapphire Rapids achieved **450 million 64-byte messages per second** using lazy flow control — updating the read pointer only every Nth message rather than per-message. Their `CLDEMOTE` instruction, which hints the hardware to push a cache line toward L3 proactively, boosted core-to-core bandwidth from **7.3 GB/sec to 22 GB/sec** when 8+ threads are involved. These are the micro-optimizations that matter after the algorithmic fundamentals are in place.

DPDK-style kernel bypass applies similar principles to network I/O: poll-mode drivers run in tight userspace loops on dedicated cores, achieving **~5 µs** average latency for UDP versus **~9 µs** through the kernel stack. The pattern translates directly to inter-thread work: dedicated polling threads, zero syscalls, pre-allocated hugepage memory, and cache-aware data layout.

---

## Cross-language performance converges at the hardware

The performance gap between languages for inter-thread communication is far smaller than most developers expect. Martin Thompson's canonical benchmark showed Java volatile ping-pong at **50 ns** versus C++ atomic at **45 ns** on identical hardware — **only ~10% difference**. Both languages emit `LOCK`-prefixed x86 instructions that bottleneck on the same cache coherence protocol. Rust's SPSC ring buffers (rtrb, crossbeam) produce essentially identical machine code to C++ equivalents and achieve comparable latencies. Rust's advantage is safety, not speed: the `Producer`/`Consumer` type split in rtrb statically prevents SPSC violations at compile time, eliminating a bug class that has plagued C++ and Java lock-free code.

Where languages do diverge is in tail latency and garbage collection. Java's GC pauses can inject multi-millisecond stalls into otherwise sub-microsecond communication paths. The Disruptor mitigated this through pre-allocated ring entries (zero allocation on the hot path), and JCTools' SPSC queues use `Unsafe`-based field access to match native performance. Go's goroutine channels are convenient but significantly slower than dedicated lock-free queues — they're designed for ease of use, not minimum latency. For Rust, the `kanal` channel recently emerged as a throughput leader in comprehensive benchmarks, though raw SPSC ring buffers remain faster for the single-producer single-consumer case.

---

## Conclusion

The fastest known inter-thread communication converges on a simple recipe: an **SPSC ring buffer** (Lamport's 1977 design, modernized with cached indices and cache-line padding) using **busy-spin waiting** on threads **pinned to cores sharing an L3 cache**. This achieves **50–135 ns round-trip latency** — within 2–5× of the bare cache coherence floor of 16–25 ns. The LMAX Disruptor proved this approach scales to complex multi-consumer topologies at 52 ns per hop in Java.

Three insights emerge that are underappreciated. First, the difference between CAS and FAA is semantic, not hardware — they cost the same per instruction (~6–7 ns uncontended), but FAA eliminates retry loops. Second, `std::mutex` is not the bottleneck most developers think: modern futex-based implementations complete 80%+ of operations without a syscall, adding only ~25 ns over raw atomics. Third, **topology awareness trumps algorithm choice** — the same queue design can vary 5× in latency depending on whether threads share an L3 (16 ns transfer) or cross an interconnect fabric (107 ns on AMD, 150 ns cross-socket on Intel). The single most impactful optimization in any inter-thread communication system is not the data structure — it is `taskset`.