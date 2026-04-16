# The fastest data structures in computer science, benchmarked

**The single biggest performance lever in modern data structure design is not algorithmic complexity — it is cache locality.** Across every category, from hash maps to graphs to heaps, implementations that minimize cache misses and exploit SIMD instructions dominate benchmarks, often by 5–20× over theoretically equivalent alternatives. This report catalogs the fastest known implementations across 10 major categories, drawing on real-world benchmarks from 2020–2026. The landscape has shifted dramatically: Google's Swiss Table design now underpins hash maps in C++, Rust, and Go; B-trees have decisively overtaken red-black trees for ordered containers; and probabilistic filters like Binary Fuse have surpassed Bloom filters in both space and speed.

---

## 1. Hash maps: SIMD probing changed everything

The modern hash map revolution began with Google's **Swiss Table** (2017), which introduced SIMD-parallel metadata probing. Every major high-performance hash map since has adopted some variant of this technique.

**boost::unordered_flat_map** (Joaquín M López Muñoz, Boost 1.81+) is now the **all-around best performer in C++** according to Jackson Allan's 2024 comprehensive benchmarks. It uses 15-element groups with an overflow byte instead of tombstones, meaning performance never degrades under heavy insert/erase workloads — a weakness of Swiss Table. SIMD matching scans all 15 reduced hash values in a single operation. It automatically applies bit-mixing for poor hash functions, making it robust against pathological inputs.

**absl::flat_hash_map** (Google Abseil) remains a top performer for large maps. It splits each 64-bit hash into H1 (57-bit group selector) and H2 (7-bit fingerprint), then broadcasts H2 across 16 metadata bytes via `_mm_set1_epi8()` and compares all 16 simultaneously with `_mm_cmpeq_epi8()`. This checks **16 slots in ~3 CPU instructions**. Elasticsearch reported **6× fewer LLC misses** after adopting a Swiss-Table-inspired design. The approach runs approximately **3× faster** than `std::unordered_map` for insertions while using 60% less memory.

**folly::F14FastMap** (Meta) takes a different architectural approach: it co-locates metadata and data within 14-entry chunks, achieving better cache behavior for small value types (sizeof < 24 bytes). F14 eliminates tombstones entirely through an overflow-bit mechanism, and Nathan Bronson (F14's author) reported it beating Swiss Table on some of Google's own benchmarks. In Rust, **hashbrown** — a direct Swiss Table port — became the standard library `HashMap` since Rust 1.36, delivering **2× speedup** over the prior Robin Hood implementation. Go 1.24 similarly adopted Swiss Table as its default runtime map.

For iteration-heavy workloads, **ankerl::unordered_dense::map** (Martin Leitner-Ankerl) stores all key-value pairs in a contiguous `std::vector`, making iteration as fast as scanning an array — unbeatable for that access pattern. For extreme load factors, **emhash** (ktprime) operates at **load factor 0.999** using 3-way linear probing with x86 bit-scan instructions to scan 64 slots at once.

For concurrent workloads, **ParlayHash** (CMU) achieves **1,130 million ops/sec at 128 threads** — an order of magnitude faster than libcuckoo (29 Mops/s) and Intel TBB (54 Mops/s) at the same thread count. In perfect hashing, **PtrHash** (Ragnar Groot Koerkamp, SEA 2025) achieves **8 ns/key** streaming throughput at 2.4 bits/key, approaching the **7.4 ns hardware limit** for random memory access.

---

## 2. B-trees have decisively beaten red-black trees

For ordered containers, the debate is settled: **B-trees outperform red-black trees by 3–18× at scale** due to cache efficiency. A `std::set<int32_t>` (red-black tree) consumes ~40 bytes per element with three pointers and a color bit per node, causing pointer-chasing cache misses on every comparison. B-trees pack dozens of keys into cache-line-sized nodes, slashing tree height by ~6×.

The **Algorithmica S+ Tree** (Sergey Slotin) achieves the most extreme results: **18× faster than `std::set`** and **7× faster than `absl::btree`** for `lower_bound` on integer keys. It uses branchless SIMD binary search within each node and compile-time constant height optimization in ~150 lines of C++. Even at node size B=32, every comparison avoids branch mispredictions entirely. **frozenca::BTree** similarly uses SIMD node searching, benchmarking insert at **175ms vs 665ms for `std::set`** on 1M int64 keys (2.6× faster lookup, 3.8× faster insert).

Google's `absl::btree_map` provides a more production-ready option with ~62 children per node for `int32_t` and 4.3–5.1 bytes/value versus 40 bytes for `std::set`. For databases, **LMDB** (Howard Chu) uses B+ trees with memory-mapped files and dominates read benchmarks against LevelDB, RocksDB, WiredTiger, and BerkeleyDB. The enhanced fork **libmdbx** claims **3× faster** than BoltDB and up to 100,000× faster in extreme cases.

The **Adaptive Radix Tree (ART)** (Viktor Leis, TU Munich, ICDE 2013) deserves special mention as the standout novel tree. It uses four adaptive node sizes (4, 16, 48, 256 children) with **SIMD-optimized search in Node16**, path compression, and lazy expansion. ART surpasses FAST trees, CSB+ trees, red-black trees, and even hash tables for main-memory database lookups, with worst-case overhead of only 52 bytes/key. It powers the HyPer database system. **HOT (Height Optimized Trie)** (Binna et al., SIGMOD 2018) further improves on ART by dynamically varying bits per node.

---

## 3. Heaps: simplicity beats theory in practice

A landmark 2014 study by Larkin, Sen, and Tarjan definitively showed that **theoretical optimality does not predict practical heap performance**. Fibonacci heaps, despite O(1) amortized decrease-key, are consistently slower than simpler alternatives due to 4 pointers per node, poor cache locality, and high constant factors.

**d-ary heaps with d=4** are the fastest for simple insert/extract-min workloads. The wider fanout (4 children vs 2) reduces tree height and cache misses, delivering **17–30% speedup over binary heaps** when the heap exceeds L1 cache. They store elements in a flat array with excellent spatial locality. When decrease-key is needed (Dijkstra's, Prim's), **pairing heaps** are the fastest pointer-based option — simpler than Fibonacci heaps with only a two-pass merge on extract-min, and "almost always faster than other pointer-based heaps including Fibonacci heaps" per the Larkin/Sen/Tarjan study.

**Brodal queues** achieve optimal worst-case bounds (O(1) everything except O(log n) delete-min) but are, in Brodal's own words, "quite complicated" and "not applicable in practice." Only one known implementation exists — a Scala functional version. For large datasets exceeding cache, **sequence heaps** (Sanders, 1999) based on k-way merging are "at least **2× faster** than optimized binary heaps and 4-ary heaps."

---

## 4. Sorting: IPS⁴o dominates parallel, pdqsort dominates sequential

**IPS⁴o (In-Place Super Scalar Samplesort)** from KIT is the clear winner across the broadest set of benchmarks. Tested across 21 sorting implementations, 6 data types, 10 distributions, 4 machines, and 7 orders of input magnitude, it outperforms the closest sequential competitor (BlockQuicksort) by **1.5×** and the closest in-place parallel competitor by **3×**. It even beats dedicated integer sorting algorithms in many configurations. Its block-based partitioning avoids branch mispredictions and achieves excellent cache behavior.

For sequential unstable sorting, **pdqsort (Pattern-Defeating Quicksort)** by Orson Peters combines randomized quicksort's average case with heapsort's worst-case guarantee while achieving linear time on sorted, reverse-sorted, and equal-element inputs. The `pdqsort_branchless` variant uses BlockQuicksort's technique to eliminate branch mispredictions on large arrays. Rust adopted pdqsort as `slice::sort_unstable`; it's also in Boost C++ Libraries.

The stable sorting landscape shifted in 2024 when Rust replaced its standard `slice::sort` with **driftsort** (Orson Peters & Lukas Bergdoll), derived from Peters' **glidesort**. Glidesort achieves up to **4× faster** than Rust's previous `slice::sort` on random data through novel "lazy logical runs," branchless bidirectional partitioning, and interleaved multi-run merging for instruction-level parallelism. Driftsort adds robustness guarantees for production use. For integer data specifically, **IPS²Ra** and **ska_sort** (Malte Skarupke) provide radix-sort alternatives; ska_sort was "used to speed things up from 3.3ms to 1.8ms in video games."

Python 3.11+ replaced Timsort's merge policy with **Powersort** for more balanced merge scheduling, while keeping the same underlying adaptive merge-sort framework that excels on partially sorted real-world data.

---

## 5. Lock-free structures: the LMAX Disruptor still reigns for latency

The **LMAX Disruptor** (Martin Thompson et al.) remains the gold standard for inter-thread communication latency. Its pre-allocated ring buffer with sequence-number-based coordination achieves **52 ns per hop** — compared to 32,757 ns for Java's `ArrayBlockingQueue`, a **630× improvement**. The C++ port reaches **50–95 million ops/sec**. The design embodies "mechanical sympathy": cache-line padding prevents false sharing, the single-writer principle minimizes contention, and pre-allocation eliminates GC pressure.

For MPMC queues, **LCRQ (Linked Concurrent Ring Queue)** by Morrison & Afek is the fastest on x86, using fetch-and-add for slot assignment with SIMD-friendly ring buffer arrays. However, it requires double-word CAS (limiting portability) and can consume ~400MB under load. The portable alternative **FAAArrayQueue** (Pedro Ramalhete) nearly matches LCRQ throughput with only 8 bytes per entry and standard CAS. For practical C++ use, **moodycamel::ConcurrentQueue** (18k+ GitHub stars) provides **1.7× faster** throughput than Boost.Lockfree through per-producer implicit sub-queues that eliminate inter-producer contention.

For lock-free stacks, the **Elimination-Backoff Stack** (Hendler, Shavit, Yerushalmi) dramatically outscales the classic Treiber stack: at 64 threads with 50/50 push/pop, it achieves **100K+ ops/msec** by matching complementary operations in a randomized elimination array, bypassing the central head-pointer bottleneck entirely.

**Read-Copy-Update (RCU)** deserves special mention for read-dominated workloads. Linux kernel RCU achieves effectively **zero read-side overhead** — no locks, no atomics, no memory barriers in optimized builds. The userspace implementation **liburcu** powers Knot DNS, GlusterFS, and ISC BIND. In Rust, **crossbeam-epoch** provides similar epoch-based reclamation and underpins most of the Rust concurrent data structure ecosystem (299M+ downloads for crossbeam-deque alone).

---

## 6. Cache-friendly design yields the largest practical speedups

**SoA (Structure of Arrays) vs AoS (Array of Structures)** is perhaps the single most impactful optimization pattern. When processing 1–2 fields across many entities, SoA delivers **1.4× to 12.7× speedup** depending on struct size and field ratio. A Go benchmark showed **100,404 ns/op for AoS vs 7,913 ns/op for SoA** (12.7× faster) when structs were large and only a subset of fields was accessed. SoA also enables direct SIMD vectorization since contiguous same-type arrays map perfectly to vector load instructions.

The **Entity Component System (ECS)** pattern in game engines operationalizes these principles at scale. **EnTT** (C++, used by Minecraft Bedrock Edition) uses sparse sets for O(1) entity-component lookup with packed arrays for iteration. **Flecs** (C/C++) and **Bevy ECS** (Rust) use archetype-based storage where entities sharing the same component set are co-located in SoA tables — delivering excellent multi-component iteration cache behavior. Archetype systems win for multi-component queries; sparse-set systems (EnTT) win for single-component queries and fast add/remove.

For memory allocation, **mimalloc** (Microsoft, Daan Leijen) **consistently outperforms** jemalloc, tcmalloc, and system malloc across diverse benchmarks — **13% faster** than tcmalloc on the Lean compiler and **15% lower P99 latency** than jemalloc for small allocations. Arena/bump allocators deliver **2–5× speedup** over general malloc for batch allocations (LLVM's `BumpPtrAllocator`, Rust's bumpalo). Google's TCMalloc achieves ~6 ns per-CPU-cache allocation at warehouse scale.

**simdjson** (Langdale & Lemire) demonstrates what SIMD-optimized data structure design can achieve: **4× faster than RapidJSON, 25× faster than JSON for Modern C++**, parsing at gigabytes/second on a single core. It processes 32–64 bytes per iteration using AVX2, with carry-less multiplication for string mask computation and ~50% fewer total instructions than scalar alternatives.

---

## 7. Tries and string structures: ART and FSTs lead

The **Adaptive Radix Tree (ART)** dominates for variable-length key indexing with its four adaptive node sizes. Node16 uses SSE instructions to compare all 16 keys simultaneously; Node48 uses a 256-byte index array for O(1) child lookup; Node256 provides direct array indexing at the cost of 2,064 bytes. Path compression and lazy expansion minimize tree height. For sorted string storage, **Tessil's HAT-trie** (C++ header-only) combines burst-trie design with cache-conscious array hash tables, using dramatically less memory than `std::unordered_map` while maintaining competitive lookup speed and sort order.

For immutable string sets, **Finite State Transducers (FSTs)** are extraordinarily space-efficient. BurntSushi's Rust `fst` crate can index **1.6 billion keys** by sharing both prefixes and suffixes (unlike tries that share only prefixes). Lucene's Java FST builds 9.8M Wikipedia terms in ~8 seconds into a 69 MB structure that can be memory-mapped for zero-copy queries. These power the term dictionaries in both Lucene and Tantivy search engines.

**JumpRope** (Joseph Gentle, Rust) claims the title of "world's fastest rope implementation" at **~35–40 million edits/second** on real editing traces — 3× faster than Ropey. It combines a skip-list spine with gap-buffer leaves. For suffix array construction, **libdivsufsort** (Yuta Mori) remains the practically fastest library despite O(n log n) worst-case complexity, beating most linear-time algorithms through excellent constant factors and cache behavior.

---

## 8. Probabilistic filters: Binary Fuse surpasses Bloom

The filter landscape has evolved significantly beyond classic Bloom filters. **Binary Fuse filters** (Thomas Mueller Graf & Daniel Lemire, 2022) achieve the best overall balance: within **13% of the information-theoretic space minimum** (vs 44% for Bloom), **2× faster construction** than Xor filters, and comparable 3-memory-access query time. They are used in production by databend, ntop, and InfiniFlow.

For raw query speed, **blocked Bloom filters** (as in Apache Impala) remain the fastest — a single cache-line-aligned memory access plus AVX2 SIMD instructions checks all hash positions simultaneously. But they pay ~44% space overhead. **Ribbon filters** (Peter Dillinger, Meta) offer the best space efficiency for write-once/read-many workloads like LSM trees: ~7 bits/key for 1% false-positive rate (vs ~10 for Bloom), integrated into RocksDB since v6.15. The **BuRR** variant achieves less than **0.5% space overhead** versus the theoretical bound.

For applications requiring deletion support, **Cuckoo filters** (Fan et al., CMU, 2014) remain the best option with constant-time 2-bucket lookups and better space efficiency than Bloom below 3% false-positive rate. **Xor filters** occupy the middle ground at 23% space overhead with very fast queries. The recommended decision framework: Binary Fuse for static sets, Cuckoo for dynamic sets needing deletion, Ribbon for LSM-tree contexts, and blocked Bloom when raw query throughput matters most.

---

## 9. Graph representations: CSR remains unbeatable for analytics

**Compressed Sparse Row (CSR)** is universally acknowledged as the fastest read-only graph representation. By storing all neighbor arrays contiguously in a single edge array with a vertex offset array, every graph access reduces to an indexed array lookup with maximal cache locality. The framework benchmarks confirm massive gaps: **graph-tool** (C++/Boost backend with OpenMP) and **NetworKit** (KIT) are **40–250× faster** than NetworkX for standard algorithms. NetworKit's PageRank on Pokec completes in **0.2 seconds** versus 59.6 for igraph.

CSR's weakness — O(n+m) cost for any edge mutation — has driven innovation in dynamic graph structures. **Terrace** (Pandey et al., 2021) uses hierarchical storage (in-place arrays, packed memory arrays, B-trees by vertex degree) to achieve **2× faster** than Aspen on graph algorithms while supporting fast batch insertions. **CSR++** (Oracle Labs) stays within **10%** of CSR's read performance while enabling **10× faster insertions**. For GPU-accelerated analytics, **Gunrock** (UC Davis) and **GraphBLAST** (Berkeley Lab) achieve order-of-magnitude speedups over CPU frameworks using frontier-based CUDA abstractions over CSR storage.

For web-scale compressed graphs, the **WebGraph framework** (Boldi & Vigna) achieves **2–6 bits per link** by exploiting power-law degree distributions, locality of reference, and the "copy property" of web graphs. **k2-trees** push this to **1.66–2.55 bits per link** for Delta-K2-tree variants, with the added advantage of supporting both forward and reverse neighbor queries.

---

## 10. Novel structures: learned indexes challenge B-trees

**Learned index structures** represent the most disruptive recent development. The **PGM-Index** (Ferragina & Vinciguerra, 2020) provides O(log log N) lookup in O(N) space using piecewise linear approximation of the data's CDF, with the 2025 **PGM++** variant achieving **2.31× speedup** over the original through mixed search strategies. **RadixSpline** (Kipf et al.) achieved **20% faster reads and 45% less memory** than B-trees when integrated into RocksDB. On the SOSD benchmark suite (200M–800M keys), learned models consistently deliver 2–3× faster lookups with smaller index sizes than traditional B-trees for static read-heavy workloads.

**Succinct data structures** trade CPU cycles for extraordinary space efficiency. The **SDSL-lite** library (Simon Gog et al.) implements highlights of 40 research publications in C++ templates, including bit vectors with rank/select (~25–50 ns), wavelet trees, FM-indexes handling inputs up to **64 GB**, and compressed suffix arrays — all near the information-theoretic space minimum.

**Cache-oblivious structures** optimize for every cache level simultaneously without knowing cache parameters. Cache-oblivious lookahead arrays (COLAs) from Bender et al. showed **790× faster** than B-trees for large random insertions. The van Emde Boas layout — recursively splitting trees and laying subtrees contiguously — achieves optimal O(log_B N) memory transfers and underpins cache-oblivious B-trees (Bender, Demaine, Farach-Colton). **Persistent data structures** also merit attention: the **CHAMP** (Compressed Hash-Array Mapped Prefix-tree) variant delivers **10–100% improvement** over classic HAMT for iteration, powering Clojure's and Scala's immutable collections. For spatial queries, **nanoflann** (C++ header-only KD-tree) achieves **50% faster queries** than FLANN through CRTP elimination of virtual dispatch and aggressive template inlining.

---

## Conclusion: the patterns that make data structures fast

Three engineering principles consistently separate the fastest implementations from the rest. First, **SIMD-parallel metadata scanning** — pioneered by Swiss Table and now used in hash maps, tries (ART Node16), Bloom filters, and B-tree node search — transforms sequential comparisons into batch operations. Second, **contiguous memory layouts** (flat arrays, SoA, packed nodes) exploit hardware prefetchers and cache lines, explaining why B-trees beat red-black trees by 18× and why flat arrays outperform linked lists by 10–100×. Third, **elimination of branches** — branchless partitioning in pdqsort/glidesort, branchless SIMD search in Algorithmica's B-tree, branchless merge loops in driftsort — avoids pipeline stalls that dominate modern CPU costs.

The practical takeaways are clear. Use **boost::unordered_flat_map** or **absl::flat_hash_map** for hash maps, **B-trees** (not red-black trees) for ordered containers, **d-ary heaps** for priority queues, **IPS⁴o** for parallel sorting, **Binary Fuse filters** for set membership, and **CSR** for graph analytics. For concurrent workloads, the LMAX Disruptor pattern (pre-allocated ring buffers with sequence numbers) and RCU (zero-cost reads) remain hard to beat. The most exciting frontier — learned indexes — is already delivering production wins in RocksDB and may fundamentally reshape database indexing within the next few years.