
**SIMD-optimized blocked Bloom filters achieve under 2.5 CPU cycles per lookup — a 22× speedup over classical implementations** — making them the fastest probabilistic membership filters available today. The performance landscape splits cleanly: blocked Bloom filters with AVX2/AVX-512 dominate raw throughput, while newer structures like binary fuse filters and ribbon filters win on space efficiency. Across languages, the gap between naïve and optimized implementations spans two orders of magnitude, from ~254 cycles/op down to ~2.5 cycles/op in C++, and from 165 seconds to 2.5 seconds for 10M insertions in Python.

The key insight from a decade of research is that **the dominant cost in Bloom filter lookups is cache misses, not computation**. Every top-performing implementation constrains operations to a single cache line, then exploits SIMD parallelism within that line. The Apache Impala split block design, created by Jim Apple, has become the de facto industry standard, deployed across Apache Arrow, Parquet, Kudu, StarRocks, and Alibaba's Hologres.

---

## SIMD blocked Bloom filters dominate raw throughput in C and C++

The progression from classical to fully optimized Bloom filters in C++ tells a dramatic story. On Intel Skylake hardware, a standard Bloom filter with k random memory accesses costs **~56 cycles/op for small filters and ~254 cycles/op for large ones** that exceed CPU cache. A cache-line blocked design drops this to a constant **~30 cycles/op** regardless of filter size. Register-blocking within a 64-bit word reaches **~20 cycles/op**. Adding AVX2 SIMD hits **~5 cycles/op**, and the ultimate patterned SIMD variant reaches **~2.5 cycles/op** — a full 22× improvement.

The **Apache Impala SimdBlockFilter** ([github.com/apache/impala](https://github.com/apache/impala)) is the most widely deployed fast implementation. Each 256-bit block (one AVX2 register) holds 8 × 32-bit words. A single hash selects the block, then 8 independent bits are set or checked simultaneously via SIMD multiply-shift hashing. The VLDB 2019 paper by Lang, Neumann, Kemper, and Boncz demonstrated that with AVX-512, batched lookups drop below **2 cycles per operation**. Their reproduction code lives at [github.com/peterboncz/bloomfilter-repro](https://github.com/peterboncz/bloomfilter-repro).

**Boost.Bloom** (new in Boost 1.89, 2025) brings production-quality SIMD Bloom filters to the mainstream C++ ecosystem. It offers multiple subfilter types — `block`, `multiblock`, `fast_multiblock32`, and `fast_multiblock64` — with AVX2, SSE2, and ARM Neon support, plus bulk operations that yield an additional **2× speedup** for batch insert/lookup. The **FastFilter/fastfilter_cpp** benchmark suite ([github.com/FastFilter/fastfilter_cpp](https://github.com/FastFilter/fastfilter_cpp), ~700 stars) is the canonical research platform, containing implementations of SIMD blocked Bloom, xor, binary fuse, cuckoo, ribbon, and Morton filters for direct comparison.

In pure C, **libbloom** ([github.com/jvirkki/libbloom](https://github.com/jvirkki/libbloom), ~1,700 stars) remains the most popular simple implementation using double-hashing with MurmurHash2. For maximum performance, Jim Apple's **libfilter** ([github.com/jbapple/libfilter](https://github.com/jbapple/libfilter)) implements the same split block algorithm used in Impala with O(1) worst-case operations.

| C/C++ Library | SIMD | Design | Performance | Stars |
|---|---|---|---|---|
| Impala SimdBlockFilter | AVX2/AVX-512 | Split block 256-bit | <2 cycles/op (batched) | ~4,400 |
| Boost.Bloom | AVX2/Neon/SSE2 | fast_multiblock64 | ~2× bulk speedup | Boost ecosystem |
| VLDB'19 bloomfilter-repro | AVX2/AVX-512 | Cache-sectorized | <2 cycles/op | ~120 |
| FastFilter benchmark suite | AVX2 | Multiple types | Research reference | ~700 |
| libbloom (C) | None | Classic double-hash | ~56–254 cycles/op | ~1,700 |

---

## Rust and Go lead the systems language tier

In Rust, **fastbloom** by tomtomwombat ([github.com/tomtomwombat/fastbloom](https://github.com/tomtomwombat/fastbloom), ~339 stars) claims the title of fastest Rust Bloom filter, reporting **2–20× speedups** over competing crates. It uses a blocked design with configurable 512/256/128/64-bit blocks, confines all operations to L1 cache, derives multiple indices from a single hash, and offers a lock-free `AtomicBloomFilter` for concurrent workloads. The **sbbf-rs** crate ([github.com/ozgrakkurt/sbbf-rs](https://github.com/ozgrakkurt/sbbf-rs)) implements the Parquet-compatible split block design with runtime AVX detection and no branching — it leads on raw member-check speed but trades higher false positive rates. For comparison, the older `bloomfilter` crate by jedisct1 benchmarks at **~17,842 ns per operation** versus newer blocked designs at **~41–72 ns**, highlighting how much architecture matters.

In Go, **blobloom** ([github.com/greatroar/blobloom](https://github.com/greatroar/blobloom)) implements cache-line-sized blocked Bloom filters and is used by Syncthing. It lets users supply pre-computed 64-bit hashes, eliminating hashing overhead from the critical path, and supports atomic concurrent adds. The venerable **bits-and-blooms/bloom** ([github.com/bits-and-blooms/bloom](https://github.com/bits-and-blooms/bloom), ~2,400 stars) is the most popular Go implementation at **~333–371 ns/op**, but **bbloom** ([github.com/AndreasBriese/bbloom](https://github.com/AndreasBriese/bbloom)) benchmarks at **68 ns/op for add and 72 ns/op for lookup** — roughly **5× faster** — by using `unsafe` and a custom fast hash function.

---

## JVM and Python: zero-allocation and native code close the gap

On the JVM, **bloom-filter-scala** ([github.com/alexandrnikitin/bloom-filter-scala](https://github.com/alexandrnikitin/bloom-filter-scala)) achieves **~11.3 million ops/sec** for lookups — **2× faster than Google Guava** at 5.7M ops/sec and 80× faster than Twitter Algebird at 157K ops/sec. The secret is **zero allocation**: while Guava creates ~1,016 bytes of garbage per `mightContain()` call (byte arrays, ByteBuffers, Hasher objects), bloom-filter-scala uses `sun.misc.Unsafe` for direct memory access, bypassing JVM array limits entirely. Guava remains the industry standard for its thread-safety (lock-free since v22.0) and battle-tested reliability. **Orestes-Bloomfilter** ([github.com/Baqend/Orestes-Bloomfilter](https://github.com/Baqend/Orestes-Bloomfilter), ~863 stars) adds Redis-backed distributed filters achieving >2M ops/sec with pipelined round-trips.

| JVM Library | Get (ops/s) | Put (ops/s) | Allocations/op | Thread-safe |
|---|---|---|---|---|
| bloom-filter-scala | ~11.3M | ~9.6M | Zero | No |
| Google Guava | ~5.7M | ~5.6M | ~1 KB | Yes (lock-free) |
| Breeze | ~5.1M | ~4.5M | ~544 B | N/A |
| Twitter Algebird | ~1.2M | ~157K | ~1.8 KB | Yes (immutable) |

In Python, the interpreter bottleneck makes native extensions essential. **rbloom** ([github.com/KenanHanke/rbloom](https://github.com/KenanHanke/rbloom), ~311 stars), written in Rust via PyO3, processes 10M insertions + 10M lookups in **2.52 seconds** on an M1 Pro — **18× faster** than pure-Python pybloom3 (46.76s) and **65× faster** than bloom-filter2 (165.54s). The C-extension **pybloomfiltermmap3** lands at 4.78 seconds with mmap-backed persistence. For Python workloads, rbloom is the clear performance winner.

---

## Binary fuse filters are the new state of the art for static sets

For immutable key sets, traditional Bloom filters are no longer optimal. The evolution from xor filters (Graf & Lemire, 2020) to **binary fuse filters** (Graf & Lemire, 2022) has produced a filter that is simultaneously smaller, faster to construct, and nearly as fast to query as blocked Bloom.

**Binary fuse 8 (3-wise)** uses just **9.0 bits per key** — only **12.5% above the information-theoretic minimum** — versus 9.84 bits for xor8 (23% overhead) and ~10 bits for standard Bloom (44% overhead). Construction runs **2× faster than xor filters**, and query speed matches xor's 3-access pattern. The 4-wise variant pushes space down to **8.6 bits/key (7.5% overhead)** at modest query cost. Reference implementations live in the FastFilter ecosystem: [github.com/FastFilter/xor_singleheader](https://github.com/FastFilter/xor_singleheader) (C single-header including `binaryfusefilter.h`), with ports to Go, Java, Zig, Rust, and Python.

**Ribbon filters** from Meta (Dillinger & Walzer, 2021) push space efficiency to the extreme: **~7 bits/key for 1% FPR** versus Bloom's 9.6 bits — a **30% space saving** — with overhead as low as **1% above theoretical minimum**. The tradeoff is 3–4× higher CPU cost per query and significant temporary memory during construction (~231 bits/key). RocksDB deploys ribbon filters for long-lived LSM-tree levels where memory savings compound, while keeping Bloom for short-lived L0 data.

**Cuckoo filters** (Fan et al., 2014) remain the best choice when deletion is required. They use ~8.5 bits/key at ≤3% FPR with only 2 memory accesses per lookup and support element removal — something standard Bloom filters cannot do. However, the VLDB 2019 paper proved that **blocked Bloom overtakes Cuckoo at high throughput** because Cuckoo touches 2 cache lines versus Bloom's 1. Cuckoo's reference C++ implementation lives at [github.com/efficient/cuckoofilter](https://github.com/efficient/cuckoofilter).

| Filter | Bits/key (1% FPR) | Space vs. theory | Query speed rank | Mutable | Deletion |
|---|---|---|---|---|---|
| Blocked Bloom (SIMD) | ~11–12 | ~60%+ over | **#1 Fastest** | Yes | No |
| Standard Bloom | ~9.6 | 44% over | Medium | Yes | No |
| Cuckoo | ~8.5 | ~6% over | Fast | Yes | Yes |
| Xor8 | 9.84 | 23% over | Fast | No | No |
| Binary Fuse 8 (3-wise) | 9.0 | **12.5% over** | Fast | No | No |
| Binary Fuse 8 (4-wise) | 8.6 | **7.5% over** | Medium-Fast | No | No |
| Ribbon | ~7.0 | **1–5% over** | Slow-Medium | No | No |

---

## The benchmark landscape and essential reading

The most authoritative benchmarks come from three sources. The **FastFilter/fastfilter_cpp** repository is the canonical multi-filter benchmark suite, maintained by Lemire, Graf, and Dillinger, covering 10+ filter types with cycle-accurate measurements on 10–100M keys. The **VLDB 2019 paper** ("Performance-Optimal Filtering: Bloom Overtakes Cuckoo at High Throughput") provides the definitive Bloom-vs-Cuckoo analysis with the critical insight that the optimal filter depends on the **ratio of lookup cost to false positive penalty**. The blog post **"Modern Bloom Filters: 22x Faster!"** ([save-buffer.github.io/bloom_filter.html](https://save-buffer.github.io/bloom_filter.html)) walks through the optimization progression step-by-step with code at [github.com/save-buffer/bloomfilter_benchmarks](https://github.com/save-buffer/bloomfilter_benchmarks).

Key academic papers worth reading, in recommended order:

- Lang et al., "Performance-Optimal Filtering: Bloom Overtakes Cuckoo at High Throughput" (VLDB 2019) — definitive throughput analysis
- Graf & Lemire, "Binary Fuse Filters: Fast and Smaller Than Xor Filters" (JEA 2022) — current state of the art for static filters
- Dillinger & Walzer, "Ribbon Filter: Practically Smaller Than Bloom and Xor" (2021) — maximum space efficiency
- Fan et al., "Cuckoo Filter: Practically Better Than Bloom" (CoNEXT 2014) — the deletion-supporting alternative
- Apple, "Split Block Bloom Filters" (arXiv 2101.01719) — the SIMD design used across Apache ecosystem

---

## Conclusion

The choice of filter depends entirely on workload characteristics. For **maximum query throughput**, SIMD blocked Bloom filters at ~2.5 cycles/op are unbeatable — choose Apache Impala's split block design in C++ or Boost.Bloom for production use. For **static/immutable sets**, binary fuse 8 filters offer the best speed-space tradeoff at just 12.5% above theoretical minimum. For **space-critical applications** at scale, Meta's ribbon filter saves 30% memory versus Bloom and is battle-tested in RocksDB. For **dynamic sets needing deletion**, cuckoo filters are the only practical high-performance option. Across managed languages, zero-allocation designs (bloom-filter-scala on JVM) and native-code extensions (rbloom for Python) close the gap to within 2–5× of C++ performance. The single most impactful optimization in any language is blocking — constraining all operations to a single cache line eliminates the dominant cost in filter lookups and delivers consistent performance regardless of filter size.

Based on the research already done, here's the definitive answer for Rust:

**For a traditional (mutable) Bloom filter, the fastest is [`fastbloom`](https://github.com/tomtomwombat/fastbloom)** by tomtomwombat. It reports 2–20× speedups over competing Rust crates, using a blocked design with configurable 512/256/128/64-bit blocks that confines all operations to L1 cache and derives multiple indices from a single hash. It also provides a lock-free `AtomicBloomFilter` for concurrent workloads, which is a big deal if you're doing anything multithreaded.

There's one caveat worth knowing: the **`sbbf-rs`** crate ([github.com/ozgrakkurt/sbbf-rs-safe](https://github.com/ozgrakkurt/sbbf-rs-safe)) implements the Parquet-compatible split block design with runtime AVX detection and no branching, and leads on raw member-check speed — but it trades higher false positive rates for that speed. So if you only care about the absolute lowest-latency `contains()` call and can tolerate more false positives, `sbbf-rs` edges ahead. For most real-world use, `fastbloom` is the better pick because it doesn't sacrifice accuracy.

**However** — if your set is **static/immutable** (built once, queried many times), you should seriously consider skipping Bloom filters entirely. Binary fuse 8 filters use just 9.0 bits per key — only 12.5% above the information-theoretic minimum — and query speed is comparable to blocked Bloom with dramatically better space efficiency. The Rust port lives in the FastFilter ecosystem (crate: `xorf`).

**TL;DR:**

| Use case                 | Crate                | Why                                        |
| ------------------------ | -------------------- | ------------------------------------------ |
| Mutable, best all-around | `fastbloom`          | Fastest with good FPR, concurrent support  |
| Raw lookup speed only    | `sbbf-rs-safe`       | Lowest latency `contains()`, higher FPR    |
| Static/immutable set     | `xorf` (binary fuse) | Smaller + fast, but no inserts after build |