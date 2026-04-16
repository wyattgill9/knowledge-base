
**No single linked list implementation dominates every scenario, but `plf::list` is the fastest general-purpose linked list in single-threaded C++, array-backed lists with perfect memory ordering can be 125× faster than naive implementations, and LCRQ achieves the highest concurrent throughput at 200+ million operations per second.** The common thread among all top performers is the same: they fight the fundamental weakness of linked lists—poor cache locality—through contiguous allocation, node unrolling, or index-based linking. This report maps the entire landscape of high-performance linked list designs, from cache-friendly variants to lock-free concurrent implementations, and identifies the top contenders in each category.

The critical insight across decades of research is that **memory layout dominates algorithmic complexity** on modern hardware. A linked list with nodes allocated sequentially in memory can outperform the same logical structure with randomly scattered nodes by two orders of magnitude. Every "fastest linked list" design exploits this principle.

## Memory layout is the single biggest performance lever

The most dramatic benchmark in linked list history comes from Johnny's Software Lab, measuring traversal of 64 million doubles across different memory layouts:

|Memory layout|Time (64M elements)|L3 data transferred|Relative speed|
|---|---|---|---|
|Random (naive malloc)|15.0 s|34.2 GB|1×|
|Compact (pool, unordered)|9.7 s|25.2 GB|1.5×|
|**Perfect (sequential order)**|**0.12 s**|**1.36 GB**|**125×**|

The random-layout list suffers a **99% L3 cache miss rate** because each pointer dereference fetches an unpredictable 64-byte cache line, of which only 8–16 bytes contain useful data. The perfect-layout list, where logical order matches physical memory order, enables hardware prefetchers to stream data efficiently—cutting transferred data by **25×**.

This gap explains why pool allocators, arena allocators, and vector-backed linked lists produce such outsized performance gains. The `jsl::vector_list` implementation stores nodes inside a `std::vector` using indices instead of pointers. Its `compact()` operation rearranges nodes to restore sequential memory ordering, achieving **7.8× faster traversal** than `std::forward_list` for 128 million integers. Even without compaction, vector-backed allocation delivers **5.25× faster traversal** and **3.2× faster insertion** than `std::forward_list`.

Intrusive linked lists push this further by eliminating separate node allocation entirely. The **Linux kernel's `list.h`**—a circular doubly-linked intrusive list using `struct list_head` embedded directly in data structures—is arguably the most battle-tested linked list in existence. Boost.Intrusive benchmarks show **5–29× faster combined insertion and destruction** versus `std::list`, because intrusive nodes share cache lines with their data and require zero heap allocations for list operations.

## `plf::list` leads the general-purpose race

Matt Bentley's `plf::list` is the current fastest drop-in replacement for `std::list` in C++. Benchmarked on Haswell CPUs with GCC 8.1, it achieves:

- **333% faster insertion** than `std::list`
- **81% faster erasure**
- **16% faster iteration**
- **77% faster sorting**
- **91% faster `remove`/`remove_if`**
- **6,500% faster `clear`** (for trivially-destructible types: **1,147,900%**)

These numbers stem from three design choices. First, `plf::list` uses **unrolled storage**: nodes live in memory blocks of up to 2,048 elements rather than individual allocations, slashing allocator overhead and improving cache locality. Second, its **spatial-aware reinsertion** mechanism finds the closest free slot to the insertion point, keeping logically adjacent elements physically nearby. Third, operations like `reverse`, `clear`, and `sort` iterate over memory blocks linearly rather than following the linked-list order—a technique impossible with `std::list`'s scattered per-node allocation.

The C++ specification itself prevents `std::list` from matching this performance. Because the standard requires O(1) `splice` for arbitrary sub-ranges, implementations cannot cluster nodes without risking pointer invalidation during partial-list splicing. `plf::list` trades away standard-compliant `splice` semantics to unlock block allocation.

Other notable single-threaded contenders include the Rust crate `orx-linked-list`, which claims **25× faster iteration** than `std::collections::LinkedList` through fragment-based contiguous storage, and Java's **GlueList**, an unrolled list that adds 1 million elements in **39.2 ms** versus LinkedList's 174.8 ms and ArrayList's 76.4 ms.

## Unrolled linked lists deliver the best cache-to-flexibility tradeoff

Unrolled linked lists store **multiple elements per node** in a small array sized to fill one or more cache lines. This single optimization reduces cache misses by a factor of _m_ (elements per node) compared to standard linked lists while preserving O(1) insertion and deletion at known positions.

Research shows unrolled lists **reduce cache misses by up to 60%** over traditional linked lists and deliver **2–4× traversal speedups** in practice. The theoretical analysis is clean: indexing requires only _n/m + 1_ cache misses versus _n_ for a standard list, approaching the optimal _n/B_ bound where _B_ is cache line capacity.

The concurrent variant is equally impressive. Platz, Mittal, and Venkatesan's **concurrent unrolled linked list with lazy synchronization** (JPDC, 2020) achieves **300% higher throughput** than other concurrent list-based sets, including Braginsky and Petrank's locality-conscious lists. Its wait-free read operations and single-node locking for most writes make it practical for real workloads. Building on this, the **DULL** (Detectable Unrolled Lock-based Linked List), published at OPODIS 2024, extends the unrolled design to persistent memory, delivering **several-fold faster performance** than competitors for update-intensive workloads while guaranteeing crash consistency.

Cache-conscious STL lists (Frias, Petit, and Roura, 2006) formalize this approach as a doubly-linked list of _buckets_, each holding Θ(B) elements tuned to cache line size. Their "2-level contiguous list" variant makes traversal performance **independent of total list size** when buckets are properly sized, because logically adjacent elements always share cache lines.

## Lock-free implementations: from Harris to LCRQ

The concurrent linked list landscape has evolved dramatically since Timothy Harris's foundational 2001 design. Harris's lock-free linked list uses **logical deletion via marked pointers** (stealing a bit in the `next` field) followed by physical unlinking—a two-step CAS protocol that remains the basis for most concurrent list implementations. Its critical weakness is that failed CAS operations trigger retraversal from the list head, making it catastrophically slow under contention on long lists.

**Träff and Pöter (2020)** addressed this with approximate backward pointers and `fetch_or`-based marking, achieving **orders-of-magnitude improvement** over textbook Harris in worst-case scenarios and meaningful throughput gains in random mixed workloads. Their six implementation variants, available at `github.com/parlab-tuwien/lockfree-linked-list`, represent the current state of the art for lock-free ordered linked lists.

For FIFO queues (the most common linked-list-based concurrent structure), the performance hierarchy is well-established:

|Algorithm|Type|Peak throughput|Key property|
|---|---|---|---|
|**LCRQ** (Morrison & Afek, 2013)|Lock-free|~200+ Mops/s|Highest throughput; uses fetch-and-add|
|**SCQ** (Nikolaev, 2019)|Lock-free|Near LCRQ|Portable, ~1 MB memory|
|**wCQ** (Nikolaev, 2022)|**Wait-free**|Matches SCQ|Per-thread progress guarantee|
|FAAArrayQueue|Lock-free|Near LCRQ|No double-width CAS needed|
|Michael-Scott Queue (1996)|Lock-free|Baseline (lowest)|Simplest, widely deployed|

LCRQ achieves its dominance through **fetch-and-add slot selection** in a ring buffer, avoiding the contended CAS retries that bottleneck the Michael-Scott queue. The tradeoff is high memory usage (~400 MB) and x86-only portability. SCQ matches LCRQ's throughput in roughly **1 MB** using single-width CAS, and wCQ adds wait-freedom with negligible overhead—making it **the fastest wait-free queue known**. These benchmarks were measured on a 72-core Intel Xeon E7-8880 v3.

Wait-free linked lists for ordered sets (Timnat, Braginsky, Kogan, and Petrank, 2012) add only **~2% overhead** versus lock-free counterparts by using a fast-path/slow-path methodology—the wait-free helping mechanism triggers in fewer than 1 in 3,000 operations. The **VBL algorithm** (Aksenov et al., PaCT 2021) is provably _concurrency-optimal_, meaning it rejects only those concurrent schedules that would violate linearizability, achieving **1.6× throughput** over Lazy Linked Lists at 72 threads.

Memory reclamation remains the dominant cost factor for lock-free structures. Epoch-based reclamation (EBR) is fastest but risks unbounded memory under stalled threads. Hazard pointers (now standardized in **C++26**) are robust but add per-access overhead. The latest advance, **Crystalline** (Nikolaev and Ravindran, PLDI 2024), achieves wait-free reclamation with high performance and bounded memory simultaneously—previously considered impossible.

## Novel designs from 2023–2026 push new boundaries

Several recent advances stand out. **Linkey** (arXiv, May 2025) is a hybrid hardware-software prefetcher specifically designed for linked data structures. Using compiler-provided hints about node layout and child pointer offsets, it achieves a **13% reduction in load misses** (up to 58.8% for some workloads) and **65.4% higher prefetch accuracy** than stride-based prefetchers—without the security vulnerabilities of Apple's content-directed prefetching approach.

**DiLi** (Ravishankar et al., 2025/2026) introduces a _distributable_ lock-free linked list that scales across machines, achieving throughput comparable to skip lists on a single node and linear scaling across distributed nodes. The **"RIP Linked List"** paper (Colnet et al., ICDSIS 2024) introduces **ArrayBlock**, a block-based array structure that outperforms all linked list variants even in benchmarks deliberately designed to favor linked lists, concluding that "it is very difficult to find much interest in using linked lists in real applications."

On the GPU side, Ashkiani et al.'s **Slab List** (IPDPS 2018) achieves **512 million updates/second** and **937 million search queries/second** on an NVIDIA Tesla K40c by organizing warp-cooperative linked lists where each "slab" fills a warp-width unit, enabling coalesced memory access—a radical rethinking of linked list design for massively parallel hardware.

## When optimized linked lists genuinely beat arrays

Despite decades of evidence favoring contiguous structures, linked lists retain genuine advantages in specific scenarios:

- **Large elements (>32 bytes):** When element size exceeds ~32 bytes, the cost of shifting array elements during mid-list insertion exceeds the cache penalty of pointer chasing. At 64 bytes per element, linked lists win for collections up to ~6,000 elements.
- **Hard real-time guarantees:** Linked list insertion/deletion at a known position is **true O(1) worst-case** with no amortized resizing spikes. This matters for avionics, game engines with 33 ms frame budgets, and audio processing.
- **Concurrent access patterns:** Lock-free linked lists and skip lists require only 1–2 local CAS operations per mutation, while arrays or balanced trees require locking or moving large regions. Java's `ConcurrentSkipListMap` scales linearly with thread count where synchronized collections collapse at 2+ threads.
- **Splice-heavy workloads:** `std::list::splice()` transfers elements between lists in **O(1)**. A real-world SAT solver showed a **5.8% slowdown** when converting from `std::list` to `std::vector` due to loss of efficient splicing.
- **Persistent/functional data structures:** Singly-linked lists enable O(1) prepend with structural sharing, providing thread safety without synchronization—the backbone of Haskell, Clojure, and Scala collections.
- **Persistent memory (PMEM):** Linked lists require fewer cache flushes per update since only modified pointers need persistence, yielding **49–81% speedups** over naive approaches when properly optimized.

Linus Torvalds's experience with software prefetching offers a cautionary tale: in Linux 2.6.40, `prefetch()` calls were **removed** from linked list traversal macros because they actively harmed performance on x86—hardware prefetchers already did a better job, and prefetching NULL at list ends caused TLB misses.

## Conclusion

The title of "fastest linked list" depends entirely on workload. For **single-threaded general use**, `plf::list` delivers the best all-around performance through unrolled block allocation and spatial-aware reinsertion, outperforming `std::list` by 3–4× on insertion and up to 6,500% on destruction. For **raw traversal speed**, vector-backed lists with perfect memory ordering (like `jsl::vector_list` with compaction) achieve the theoretical maximum—**125× faster** than randomly allocated lists. For **concurrent throughput**, LCRQ and its wait-free descendant wCQ dominate queues at 200+ Mops/s, while Träff-Pöter's improved Harris list and VBL represent the frontier for ordered concurrent sets. For **systems programming**, Linux kernel-style intrusive lists remain unmatched in their zero-allocation, zero-overhead elegance.

The deeper insight is that the "fastest linked list" increasingly doesn't look like a linked list at all. The winning designs—unrolled lists, vector-backed lists, slab lists—are hybrids that borrow the contiguous memory layout of arrays while preserving the O(1) pointer-manipulation semantics of linked structures. The field is converging on a principle that Bjarne Stroustrup articulated in 2012 and that every subsequent benchmark has confirmed: **on modern hardware, cache locality trumps algorithmic complexity for virtually all practical collection sizes**.