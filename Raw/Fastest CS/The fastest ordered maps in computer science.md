
**SIMD-optimized B-trees are the fastest general-purpose ordered maps on modern hardware, outperforming red-black trees by 7–18× for lookups.** For dense integer keys, Adaptive Radix Trees (ART) match or exceed even tuned B-trees by eliminating key comparisons entirely. The practical winner depends on workload: read-only integer searches favor SIMD B-trees, general-purpose production code favors Google's Abseil `btree_map`, and concurrent in-memory databases favor ART with optimistic lock coupling or Masstree. The era of the pointer-chasing binary search tree — still the default in `std::map`, Java's `TreeMap`, and most standard libraries — is definitively over for performance-critical applications.

The root cause is simple: a single cache miss to DRAM costs **60–100 ns** (150–300 CPU cycles), while an L1 cache hit costs **~1 ns**. Red-black trees chase one pointer per comparison, hitting DRAM at nearly every level. B-trees pack 15–64 keys per cache line, collapsing a 20-level binary tree into 3–4 levels. SIMD instructions then search each node in a single clock cycle. This **cache × SIMD** combination creates an order-of-magnitude advantage that no amount of algorithmic cleverness in binary trees can overcome.

## Theoretical bounds set the ceiling but not the winner

The comparison-based lower bound of **Ω(log n)** per operation applies to any structure that only compares keys. Red-black trees, AVL trees, B-trees, skip lists, treaps, and splay trees all achieve this bound — they differ only in constants, guarantees (worst-case vs. amortized), and practical cache behavior.

Three structures beat O(log n) by exploiting integer key structure:

- **Van Emde Boas trees** achieve **O(log log U)** per operation where U is the universe size. For 64-bit integers, this is roughly 6 operations — impressive, but the O(U) space requirement is prohibitive without hashing, and the constant factors are large. Practical implementations exist but rarely outperform tuned B-trees on real hardware.

- **Fusion trees** achieve **O(log n / log log n)** using word-level parallelism to compare multiple keys per machine word. With 64-bit words, this means roughly a 4× reduction in tree height compared to binary search. However, the bit manipulation required is so complex that **no practical implementation competes** with straightforward B-trees.

- **Exponential search trees** (Andersson & Thorup) push the theoretical frontier to **O(√(log n / log log n))** worst-case deterministic — essentially O(√log n). Pătrașcu and Thorup proved this is near-optimal for linear-space predecessor structures. These remain purely theoretical.

In the external memory model, **B-trees achieve optimal O(log_B N) I/Os** per operation. **Bε-trees** (fractal tree indexes) trade slightly slower reads for dramatically faster writes: insertion costs only O(log_B N / B^(1−ε)) amortized I/Os, which for typical parameters is **32× fewer I/Os than B-trees** for writes. TokuDB used this in production until its deprecation in 2022, and the principle lives on in LSM-tree variants. Cache-oblivious B-trees match B-tree bounds without knowing hardware parameters, but tuned cache-conscious designs consistently outperform them in practice.

## SIMD B-trees dominate single-threaded benchmarks

The practical performance hierarchy for ordered maps on modern hardware, measured on integer keys at scale (10⁵–10⁷ elements), is remarkably clear:

**Tier 1 — SIMD-optimized B-trees (7–18× faster than `std::map`):** The implementation described on Algorithmica (by Sergey Slotin) uses AVX2 for branchless intra-node search, compile-time-constant tree heights, and hugepage allocation. Against `std::set` (red-black tree), it achieves **7–18× speedup on `lower_bound`** and **3–8× on `insert`**. Against Abseil's `btree_set`, it still wins **3–7× on lookups**. The entire implementation is under 150 lines of C++. The recent BS-tree (ICDE 2026) takes this further with AVX-512, processing **16 keys per node using two 512-bit SIMD instructions**, with gapped nodes enabling branchless updates. The FB+-tree (VLDB 2025) extends SIMD B-trees to variable-length keys using byte-wise prefix matching with AVX-512.

**Tier 2 — Specialized B-tree libraries (2–4× faster than `std::map`):** The `frozenca::BTreeSet` (C++20, SIMD-enabled) benchmarks at **175 ms** for 1M random int64 inserts vs. ~498 ms for `std::set` — roughly **2.8× faster for inserts** and **4× faster for lookups**. It also beats Google's cpp-btree by 1.4–1.6×. The `kressler/fast-containers` B+Tree claims **2–5× faster than Abseil** with hugepage allocation providing 3–5× speedup alone for working sets exceeding 1 GB.

**Tier 3 — Production B-tree maps (1.5–3× faster than `std::map`):** Google's Abseil `btree_map` is the gold standard for production use. It uses **~5 bytes per element** vs. 40 bytes for `std::set<int32_t>` — an **8× memory reduction**. Tree height drops from ~20 levels (red-black) to ~3–4 levels (B-tree with fanout 62), yielding **~6× fewer cache misses**. The TLX B+-tree (Timo Bingmann) becomes significantly faster than `std::map` above 16,000–100,000 items depending on hardware.

**Tier 4 — Standard library defaults (baseline):** `std::map` (red-black tree) serves as the baseline. Its per-node heap allocation, 40-byte overhead per element, and single-key-per-cache-line layout make it the slowest mainstream ordered map by a wide margin on modern hardware.

| Factor | Red-black tree (`std::map`) | Abseil `btree_map` | SIMD B-tree (Algorithmica) |
|--------|---------------------------|--------------------|-----------------------------|
| Tree height (1M int32 keys) | ~20 levels | ~3–4 levels | ~3 levels |
| Memory per element | 40 bytes | ~5 bytes | ~5.5 bytes |
| Cache misses per lookup | ~20 | ~4 | ~3 |
| Intra-node search | scalar, 1 key | linear, ~30 keys | **SIMD, 8+ keys/cycle** |
| Branch mispredictions | ~10 per lookup | ~3–4 per lookup | **~0 (branchless)** |

## ART and tries win for specific key distributions

Adaptive Radix Trees flip the script on B-trees for certain workloads. Instead of comparing keys, ART decomposes keys byte-by-byte through a trie with four adaptive node types: **Node4** (linear scan), **Node16** (SSE2 parallel compare of all 16 keys in one instruction), **Node48** (256-byte index array), and **Node256** (direct array access). This gives **O(k) lookup time** independent of the number of elements — for 8-byte integer keys, that's a fixed 8 byte lookups regardless of whether the tree holds 1,000 or 1 billion entries.

For dense integer keys, ART **outperforms even FAST** (the SIMD-optimized static search tree from Intel) and approaches hash table speeds while preserving sorted order. The ART paper (Leis et al., ICDE 2013) showed it matching hash tables for point queries with only **8.1 bytes per key** overhead for dense datasets. **Height Optimized Tries (HOT)** extend this advantage to string keys by dynamically varying bits per node based on data distribution, outperforming ART, B-trees, and Masstree for string workloads.

The Cuckoo Trie (2021) pushes further by exploiting **memory-level parallelism** — modern out-of-order CPUs can execute multiple independent DRAM accesses simultaneously, but pointer-chasing trees serialize these accesses. By restructuring lookups to issue independent memory accesses, the Cuckoo Trie **outperforms state-of-the-art indexes by 20–360%** with a smaller memory footprint.

Where B-trees still win: ART struggles with long string keys sharing common prefixes (each trie level may trigger a cache miss for a single byte of discrimination) and range scans (trie iteration is inherently less cache-friendly than scanning B-tree leaves). For **general-purpose ordered maps with arbitrary key types**, B-trees remain the better default.

## Concurrent ordered maps: optimistic locking beats lock-freedom

The definitive benchmark comparison comes from the OpenBw-Tree paper (Wang et al., SIGMOD 2018), which tested five concurrent ordered map implementations head-to-head. The results overturned conventional wisdom: **the lock-free Bw-tree was 1.5–4.5× slower than lock-based alternatives**. The ranking:

1. **ART with Optimistic Lock Coupling (OLC)** — fastest overall for point operations. Readers never acquire locks or write shared memory, eliminating cache-line invalidation across cores. The Congee library (Rust ART-OLC) achieves **150 million ops/sec on 32 cores** for 8-byte keys. ART-OLC requires **4.8× fewer instructions and 3.6× fewer L3 cache misses** than B+Trees with traditional lock coupling.

2. **Masstree** — best under high contention and for variable-length string keys. Its "cache craftiness" principles ensure read operations never dirty shared cache lines. It achieves **6–10 million ops/sec on 16 cores** for YCSB workloads, only 2.5× slower than a near-optimal hash table — a small price for range query support. Masstree's trie-of-B+-trees architecture slices keys into 8-byte chunks, combining trie efficiency for long keys with B+-tree cache locality.

3. **B+Tree with OLC** — dominant for range scans. The BP-Tree (VLDB 2023) achieves **7.4× faster than Masstree** and **30× faster than Masstree** on large range queries, while matching Masstree on point operations. B-skiplists (ICPP 2025) achieve **2–9× higher throughput** than traditional concurrent skip lists by packing multiple elements per node.

4. **Java's ConcurrentSkipListMap** — the simplest correct concurrent ordered map. Single-threaded it loses to `TreeMap`, but it scales well under contention. Bronson's SnapTree (concurrent AVL) showed **39% higher throughput** than ConcurrentSkipListMap in benchmarks.

The core insight: **minimizing cache misses and cache-line invalidation matters more than eliminating locks**. The Bw-tree's delta chains cause additional pointer chasing; its CAS-heavy design triggers cache coherence traffic. Meanwhile, optimistic readers in ART-OLC and Masstree perform zero writes to shared memory, achieving near-lock-free scalability with dramatically simpler code.

## The frontier: AVX-512, learned indexes, and memory-level parallelism

Three developments from 2024–2026 are reshaping the landscape. First, **AVX-512 is now ubiquitous** — AMD Zen 5 (2024) provides native 512-bit execution (not double-pumped), and Intel's Granite Rapids delivers full AVX-512 throughput. This enables B-tree nodes with **16 × 32-bit keys searched in two instructions**, as demonstrated by the BS-tree. Second, **learned indexes** replace tree levels with lightweight ML models that predict key positions. LITS (2024) combines learned models with tries, claiming **2.4× improvement over HOT** for point operations. Third, AMD's **3D V-Cache** technology pushes L3 caches to 96–128 MB, dramatically raising the threshold at which tree indexes spill to DRAM — a B-tree with fanout 64 can hold **billions of keys** entirely in L3 cache on these processors.

## Conclusion

**For the single fastest ordered map implementation today:** if your keys are integers and your workload is read-heavy, a hand-tuned **SIMD B-tree with AVX-512 and hugepages** (like the Algorithmica design or BS-tree) is the fastest known structure, achieving lookup throughputs of **50–100+ million queries/second** on a single core. If you need a production-ready drop-in replacement, **Abseil `btree_map`** delivers 1.5–3× over `std::map` with zero tuning. For dense integer keys or in-memory databases, **ART** matches hash table speeds while preserving order. For concurrent workloads, **ART with optimistic lock coupling** or **Masstree** represents the state of the art.

The unifying principle across all winners is identical: **pack more keys per cache line, search them with SIMD, and eliminate branch mispredictions**. The theoretical sub-logarithmic structures (van Emde Boas, fusion trees) remain curiosities — their constant factors drown in the 100-ns reality of a single cache miss. On modern hardware, the memory hierarchy is the algorithm.