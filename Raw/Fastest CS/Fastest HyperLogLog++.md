# Cloudflare's Rust HLL dominates single-threaded throughput

**The fastest production-quality HyperLogLog implementation is Cloudflare's `cardinality-estimator` Rust crate**, achieving **2.7 ns per insert (~370 million inserts/sec)** at 1M-element cardinality on an Intel i7-13800H — including hashing. It also delivers the fastest estimation calls at a constant **0.28 ns regardless of cardinality**, an O(1) operation that no other implementation matches. For pure peak insert speed at high cardinalities, the Rust `hyperloglogplus` crate edges it out at **2.05 ns/insert (~488M inserts/sec)**, but this comes with catastrophic mid-range slowdowns that make it impractical as a general-purpose choice. For SIMD-accelerated merge and count operations, **Valkey's AVX2-optimized Redis fork delivers a 12–13× speedup**, the single largest documented SIMD gain for HLL operations.

No comprehensive cross-language benchmark suite exists for HyperLogLog. The numbers below are synthesized from independent benchmarks on different hardware — treat cross-language comparisons as directional rather than precise.

## The definitive insert throughput rankings

The most rigorous head-to-head benchmark data comes from the `cardinality-estimator-safe` project, which tested six Rust HLL crates on identical hardware (i7-13800H, `target-cpu=native`, precision p=12, 6-bit registers). All measurements include hashing (WyHash for Cloudflare's crate, SipHash for amadeus, Murmur-family for others).

| Implementation | Language | ns/insert at 1M | ns/insert at 4K | Inserts/sec (1M) | Consistent? |
|---|---|---|---|---|---|
| **hyperloglogplus (tabac)** | Rust | **2.05** | 34.5 | ~488M | No — 92 ns at 1K |
| **cardinality-estimator** | Rust | **2.70** | 3.95 | ~370M | Yes — 2.7–5.9 ns across all ranges |
| **cardinality-estimator-safe** | Rust | **2.67** | 6.70 | ~375M | Yes — safe Rust, no `unsafe` |
| **amadeus-streaming** | Rust | 4.96 | 5.05 | ~200M | Yes — flat ~5 ns everywhere |
| **hyperloglog (jedisct1)** | Rust | 5.05 | 7.91 | ~200M | Moderate variance |
| **DataSketches HLL_8** | Java | ~10.7 | ~10.7 | ~93M | Yes — flat asymptote |
| **DataSketches CPC** | Java | ~10.3 | ~10.3 | ~97M | Yes |
| **Clearspring HyperLogLogPlus** | Java | ~105 | ~105 | ~9.5M | Yes — consistently slow |

Cloudflare's crate uses **WyHash** (one of the fastest non-cryptographic hashes at ~1–2 ns), an adaptive three-representation storage model (Small → Array → HyperLogLog), and compiler auto-vectorization via `target-cpu=native`. It does not use explicit SIMD intrinsics. The **~2–3 ns range appears to be the practical floor** for HLL insertion including hashing on modern x86 hardware, since the hash function alone costs 1–2 ns and the register update adds another 1–2 ns.

The tabac `hyperloglogplus` crate's 2.05 ns peak is misleading as a general recommendation. Its sparse-to-dense transition creates **92 ns/insert at cardinality 1,024** — a 45× slowdown — and its heap usage balloons to **195,831 bytes** versus Cloudflare's **4,092 bytes**. For any workload where cardinality isn't guaranteed to be massive, it's a poor choice.

## Estimation and merge speeds reveal a wider gap

Estimation speed is where Cloudflare's implementation truly separates from the pack. By maintaining a running harmonic sum and zero-register count during each insert, it achieves **O(1) estimation in 0.28 nanoseconds** — effectively free. Every other tested implementation requires iterating all registers, yielding estimation times that scale with precision:

| Implementation | Estimate at 1M cardinality |
|---|---|
| **cardinality-estimator (Cloudflare)** | **0.28 ns** |
| amadeus-streaming | 2.23 ns |
| hyperloglog (jedisct1) | 7,432 ns |
| hyperloglogplus (tabac) | 2,191 ns |
| probabilistic-collections | 7,734 ns |

For **merge operations**, Apache DataSketches HLL_8 (Java) completes a sketch-to-sketch merge in **4,399 ns** (~4.4 µs), while HLL_4 takes **11,545 ns**. DataSketches reports a merge throughput of up to **14.5 million sketches merged per second per thread** for their optimized paths. The Clark Duvall Go implementation benchmarks at ~36,267 ns per merge — roughly 8× slower than DataSketches HLL_8.

DataSketches excels at **serialization/deserialization**: HLL_4 serializes in **472 ns** and deserializes in **150 ns**, compared to Clearspring's ~100,000 ns range. Off-heap mode effectively eliminates deserialization cost entirely via pointer wrapping (~100 ns).

## SIMD and hardware acceleration change the merge/count calculus

The most impactful SIMD optimization found targets **merge and count operations, not insertion**. The core HLL insert — hash a value, extract register index, compute leading zeros, compare-and-max — is inherently scalar and doesn't benefit much from vectorization. But scanning 16,384 packed 6-bit registers for count or merge is a perfect SIMD target.

**Valkey's AVX2 optimization** (merged into Valkey 8.1, proposed for Redis via issue #13551) delivers the most dramatic SIMD gains documented for any HLL implementation:

| Operation | Scalar (ops/sec) | AVX2 (ops/sec) | Speedup | Hardware |
|---|---|---|---|---|
| PFCOUNT | 5,281 | **69,803** | **13.2×** | i9-13900H |
| PFMERGE | 9,446 | **120,642** | **12.8×** | i9-13900H |
| PFMERGE (NEON) | 7,420 | **76,980** | **10.4×** | AWS Graviton 3 |

These are server-level benchmarks including network overhead. The technique works by vectorizing the conversion between Redis's 6-bit packed dense encoding and full-byte register arrays. AVX2 processes **32 bytes (42+ registers) per iteration**; ARM NEON processes 16 bytes per iteration. The p50 latency for PFCOUNT dropped from ~35 ms to **2.75 ms** with AVX2.

**Daniel Baker's `sketch` library** (C++, header-only) provides SIMD-accelerated HLL with SSE2, AVX2, and AVX-512BW support. While no standalone ns/insert microbenchmarks are published, the Dashing genomics tool built on this library is **3.5–4.7× faster** than competitors (Mash, BinDash) in the sketch-building phase across 87,113 genomes. AVX-512BW provided a **~20% speedup** in the JMLE (Joint Maximum Likelihood Estimation) inner loop over non-AVX-512 builds.

For **GPU acceleration**, Bozkus and Fraguela (2017) demonstrated an **88.6× speedup** over optimized sequential HLL on an NVIDIA Tesla K20m using OpenCL with fine-grained locking and speculative checking. On FPGA, Kulkarni et al. (2020) achieved **2.6× higher throughput** than CPU/GPU implementations using an "optimistic sketching architecture" that partitions registers into independent banks. Neither approach has resulted in a widely available library.

## Concurrent and lock-free implementations for multi-threaded workloads

The `hyperloglockless` Rust crate provides the best concurrent HLL story. Its `AtomicHyperLogLog` uses **lock-free atomic operations** — all methods take `&self` rather than `&mut self`, eliminating the need for `RwLock` or `Mutex` wrappers. It claims **5× faster sparse insertion** and **~100× better accuracy** than other sparse HLL implementations, though exact ns/insert figures are only available as benchmark chart images in the README, not numeric tables.

For server-oriented concurrent access, `hlld` (C, by Armon Dadgar) operates as a daemon achieving **1M+ operations per second** on a basic MacBook Air, handling HLL operations over an ASCII protocol with multi-threaded support.

In the database world, **ClickHouse** uses an Ertl (2017)-based HLL (`uniqHLL12`) with 5-bit cells in just ~2.5 KB, integrated into its vectorized columnar execution engine. It's reported as **25–30× faster** than `COUNT(DISTINCT ...)` with ~99.8% accuracy. DuckDB and Presto/Velox (Meta) also have internal HLL implementations, but neither publishes raw throughput benchmarks.

## Choosing the right implementation depends on your bottleneck

The answer to "what is the single fastest HLL" depends on which operation dominates your workload:

- **Insert-dominated, single-threaded**: Cloudflare's `cardinality-estimator` (Rust) — **2.7 ns/insert, 0.28 ns estimate, 4 KB memory**. Best all-around choice. Uses WyHash, adaptive representations, and compiler auto-vectorization.
- **Insert-dominated, high cardinality only**: `hyperloglogplus` (Rust, tabac) — **2.05 ns/insert at 1M cardinality**, but avoid if cardinalities vary.
- **Merge/count-dominated**: Valkey with AVX2 — **12–13× faster** merge and count. If building a custom system, apply the same SIMD 6-bit unpacking technique to any implementation.
- **Concurrent multi-threaded**: `hyperloglockless` (Rust) — lock-free atomics, O(1) estimation, no contention.
- **JVM ecosystem**: Apache DataSketches HLL_8 — **10.7 ns/insert**, the gold standard in Java, 10× faster than Clearspring's HyperLogLogPlus.
- **Maximum parallel throughput**: GPU via OpenCL (88.6× demonstrated) or FPGA (2.6× over CPU/GPU) for bulk processing at massive scale.

## Conclusion

The ~2–3 ns/insert floor for single-threaded HLL insertion on x86 is fundamentally constrained by hash function latency. Cloudflare's Rust crate reaches this floor while maintaining consistent performance across all cardinality ranges, O(1) estimation, and minimal memory — making it the best general-purpose choice. The Rust ecosystem has pulled decisively ahead of Java (DataSketches at 10.7 ns) and C (Redis at ~5+ ns raw), largely thanks to WyHash's speed, zero-cost abstractions, and aggressive compiler optimizations. No implementation uses explicit AVX2/AVX-512 intrinsics for the insert path because it's inherently scalar; SIMD's real payoff is in merge/count operations, where Valkey's 12× acceleration sets the benchmark. For throughput beyond what a single core can deliver, lock-free atomic implementations like `hyperloglockless` scale across threads without contention, while GPU and FPGA approaches offer order-of-magnitude gains for batch processing at the cost of integration complexity.