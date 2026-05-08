---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Fastest Linked Lists

The deeper truth across the whole landscape: the fastest "linked list" implementations don't really look like linked lists anymore. The winners — [[plf-list]], [[unrolled-linked-list|unrolled lists]], [[vector-backed-list|vector-backed lists]], [[slab-list]] — borrow contiguous memory layout from arrays while keeping the O(1) pointer-manipulation semantics that linked lists were originally chosen for. The 2024 "RIP Linked List" paper put it bluntly after benchmarking: their **ArrayBlock** structure beats every linked-list variant *even on benchmarks deliberately designed to favor linked lists*. On modern hardware, cache locality trumps algorithmic complexity for virtually all practical collection sizes.

## The 125× memory-layout result

The single most striking benchmark in linked-list research, from Johnny's Software Lab — traversal of 64 million doubles:

| Layout | Time | L3 transferred | Speed |
|--------|------|----------------|-------|
| Random (naive `malloc`) | 15.0 s | 34.2 GB | 1× |
| Compact (pool, unordered) | 9.7 s | 25.2 GB | 1.5× |
| **Perfect (sequential)** | **0.12 s** | **1.36 GB** | **125×** |

Random layout suffers a **99% L3 miss rate** because each pointer dereference fetches a 64-byte cache line of which only 8–16 useful bytes are touched. Perfect layout streams through hardware prefetchers, transferring 25× less data. The same logical structure differs by two orders of magnitude in throughput depending purely on where the nodes sit in memory.

This single result explains the entire ecosystem of "fast linked list" designs: every one of them is an answer to the question "how do we make node memory layout match traversal order without giving up the O(1) operations that motivated using a linked list in the first place?" The same mechanical-sympathy lesson runs through the [[fastest-ordered-maps|ordered maps]], [[fastest-hash-map-2025|hash maps]], and [[fastest-dynamic-arrays|dynamic arrays]] hierarchies — see [[mechanical-sympathy]].

## The single-threaded leader: plf::list

[[plf-list|Matt Bentley's plf::list]] is the current fastest drop-in `std::list` replacement in C++. Benchmarked on Haswell with GCC 8.1:

| Operation | vs `std::list` |
|-----------|----------------|
| Insertion | 333% faster |
| Erasure | 81% faster |
| Iteration | 16% faster |
| Sorting | 77% faster |
| `remove` / `remove_if` | 91% faster |
| `clear` | **6,500% faster** (1,147,900% for trivially-destructible) |

Three design choices: **unrolled storage** in 2,048-element blocks, **spatial-aware reinsertion** that places new nodes near their neighbors, and **block-linear bulk operations** for `reverse`/`clear`/`sort`. Notably, the C++ standard *prevents* `std::list` from matching this — its O(1) `splice` requirement means implementations cannot cluster nodes safely. Performance vs. standard-compliance trade-off, made explicit.

## The unrolled tier

[[unrolled-linked-list|Unrolled linked lists]] store multiple elements per node in a small array sized to fill one or more cache lines. Cache misses drop by ~60%; traversal is 2–4× faster than standard linked lists; indexing requires only n/m + 1 misses versus n. The **concurrent unrolled list with lazy synchronization** (Platz, Mittal, Venkatesan, JPDC 2020) achieves **300% higher throughput** than other concurrent list-based sets, including Braginsky-Petrank's locality-conscious lists. Building on this, **DULL** (OPODIS 2024) extends the design to persistent memory.

[[vector-backed-list|Vector-backed lists]] take this further. `jsl::vector_list` stores nodes inside a `std::vector` using indices instead of pointers, with a `compact()` operation that restores sequential ordering — **7.8× faster traversal** than `std::forward_list`. Rust's `orx-linked-list` claims **25× faster iteration** than `std::collections::LinkedList` via fragment-based contiguous storage. Java's GlueList adds 1M elements in 39.2 ms vs. 174.8 ms for `LinkedList`.

## The intrusive tier

[[intrusive-list|Intrusive linked lists]] eliminate separate node allocation entirely — the list pointers live inside the data structures being linked. The **Linux kernel's `list.h`** (a circular doubly-linked intrusive list using `struct list_head` embedded in data structures) is arguably the most battle-tested linked list in existence. **Boost.Intrusive** benchmarks show 5–29× faster combined insertion and destruction versus `std::list` because intrusive nodes share cache lines with their data and require zero heap allocations.

Linus Torvalds removed `prefetch()` from the kernel list macros in 2.6.40 because hardware prefetchers were already doing better — and prefetching the NULL at list ends caused TLB misses. A cautionary tale for hand-rolled prefetch.

## The concurrent landscape

The lock-free linked list lineage starts with **Timothy Harris (2001)** — [[harris-linked-list|the marked-pointer logical-deletion design]] still underlies most modern concurrent lists. Its weakness is failed-CAS retraversal from the head, catastrophic on long lists. **Träff-Pöter (2020)** addressed this with approximate backward pointers and `fetch_or`-based marking, achieving orders-of-magnitude improvement in worst-case scenarios. **VBL** (Aksenov et al., PaCT 2021) is provably *concurrency-optimal* — 1.6× over Lazy Linked Lists at 72 threads.

For FIFO queues (the most common linked-list-based concurrent structure), the modern hierarchy from the [[the-fastest-queue|queues survey]]:

| Algorithm | Type | Peak throughput |
|-----------|------|-----------------|
| [[lcrq\|LCRQ]] | Lock-free | ~200+ Mops/s |
| [[scq\|SCQ]] | Lock-free | Near LCRQ; portable; ~1 MB |
| [[wcq\|wCQ]] | **Wait-free** | Matches SCQ |
| FAAArrayQueue | Lock-free | Near LCRQ |
| [[michael-scott-queue\|Michael-Scott]] (1996) | Lock-free | Baseline |

These are the same designs that win the queues survey and are precisely the place where the "linked list" abstraction was abandoned in favor of FAA-on-a-ring buffers. The underlying lesson — [[faa-vs-cas|FAA-over-CAS]] at contention hotspots — is the same one that produces the modern concurrent ordered map hierarchy via [[art-olc|ART-OLC]] and [[masstree]] in [[fastest-ordered-maps]].

## Memory reclamation: the dominant cost

For lock-free linked structures, the safe-to-reclaim problem dominates throughput. The 2025 hierarchy:

- [[crossbeam-epoch|Epoch-based reclamation (EBR)]] — fastest in the common case; risks unbounded memory growth under stalled threads.
- [[hazard-pointers|Hazard pointers]] — robust, bounded memory; per-access overhead. **Standardized in C++26.**
- [[crystalline-reclamation|Crystalline]] (Nikolaev & Ravindran, PLDI 2024) — wait-free reclamation with bounded memory, **previously considered impossible**.

This is the same reclamation problem the [[bw-tree]]'s delta chains and the lock-free queue family run into; Crystalline's result genuinely shifts the design space.

## GPU: warp-cooperative slabs

[[slab-list|Slab Lists]] (Ashkiani et al., IPDPS 2018) reorganize linked lists to fit GPU execution: each "slab" fills a warp-width unit and is processed by 32 threads cooperatively, enabling coalesced memory access. **512 million updates/second** and **937 million search queries/second** on a Tesla K40c. The same warp-centric redesign principle is what produces the [[gpu-queues|fastest GPU queues]] like [[bacq]].

## When linked lists genuinely win

Despite the 125× layout penalty, linked lists retain real advantages in narrow cases:

- **Large elements (>32 bytes)** — past the threshold, mid-list array shifts cost more than pointer chasing. At 64 bytes per element, linked lists win for collections up to ~6,000 elements.
- **Hard real-time guarantees** — true O(1) worst-case insert/delete with no amortized resizing spikes. Avionics, game-engine 33 ms frame budgets, audio DSP.
- **Splice-heavy workloads** — a real SAT solver measured a **5.8% slowdown** when converting `std::list` to `std::vector` for the splice cost. `std::list::splice` is genuinely O(1) and array-backed structures cannot match it.
- **Concurrent fine-grained mutation** — lock-free linked lists and skip lists need only 1–2 local CAS operations per mutation. Java's `ConcurrentSkipListMap` scales linearly while synchronized collections collapse at 2+ threads.
- **Persistent / functional data structures** — [[persistent-functional-list|singly-linked `cons` lists]] enable O(1) prepend with structural sharing, providing thread safety without synchronization. The backbone of Haskell, Clojure, and Scala collections.
- **Persistent memory (PMEM)** — fewer cache flushes per update since only modified pointers need persistence. **49–81% speedups** over naive approaches.

For everything else, prefer contiguous structures — and when you do reach for a "linked list," reach for one of the hybrids ([[plf-list]], [[unrolled-linked-list]], [[vector-backed-list]], [[intrusive-list]]) rather than `std::list` or `std::collections::LinkedList`.

## The 2024–2026 frontier

**Linkey** (arXiv, May 2025) is a hybrid hardware-software prefetcher specifically designed for linked data structures. Using compiler-provided hints about node layout and child-pointer offsets, it achieves a 13% reduction in load misses (up to 58.8% in some workloads) and 65.4% higher prefetch accuracy than stride-based prefetchers — without the security vulnerabilities of Apple's content-directed prefetching approach.

**DiLi** (Ravishankar et al., 2025/2026) introduces a *distributable* lock-free linked list that scales linearly across machines, achieving throughput comparable to skip lists on a single node. The frontier is no longer "make linked lists faster" — it's "make linked-shaped abstractions scale across cache hierarchies and across machines."

## The unifying principle

Every winner in this hierarchy fights the same fight: **make node memory layout match access order**. Block-allocated nodes ([[plf-list]]), array-based nodes ([[vector-backed-list]]), embedded nodes ([[intrusive-list]]), warp-aligned slabs ([[slab-list]]), or ring-buffer-shaped queues ([[lcrq]]). The "linked list" name is increasingly archaeological. The structures that survived are the ones that gave up the textbook linked-list memory model and kept only the parts that hardware actually rewards.
