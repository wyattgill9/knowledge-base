---
tags:
  - performance
  - concurrency
  - architecture
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# False Sharing and the 128-Byte Rule

False sharing is what happens when two independent variables touched by different threads occupy the same cache line: every write by one thread invalidates the other thread's cache copy, even though the threads aren't actually sharing data. The standard fix — pad to 64 bytes (one cache line) — is **wrong on modern x86**. The right answer is **128 bytes**, because the *adjacent cache-line prefetcher* speculatively pulls in the neighboring line on every fetch. Padding to 128 bytes between contended fields delivers up to **3.1× over unpadded queues** in benchmarks.

## What false sharing costs

When two independent variables share a 64-byte cache line:

- Thread A writes its variable → CPU acquires the cache line in Modified state, invalidating B's copy.
- Thread B reads its variable → CPU must fetch the line from A's L1, costing 50–200 ns of [[cache-coherency|MESI traffic]].
- Repeat per operation.

Throughput can drop **10× or more** even though the threads are nominally independent. The classic case is a producer's tail index and a consumer's head index sharing a cache line in a [[ring-buffer]].

## Why 64 bytes is no longer enough

Modern Intel and AMD CPUs ship an *adjacent cache-line prefetcher* (also called the "L2 streamer" or "spatial prefetcher") that, on every fetch, speculatively loads the *paired* cache line — the 64-byte line adjacent to the requested one, aligned to a 128-byte boundary. The intent is to exploit spatial locality: if you read `data[0..63]`, you probably want `data[64..127]` next.

For false sharing, this is catastrophic. Even if you carefully separate two contended variables onto different 64-byte cache lines, the prefetcher pulls both into the requesting core's cache, recreating the false-sharing problem one layer down.

The fix is to align contended variables to **128-byte boundaries** so the prefetched neighbor of each one is empty padding rather than the other variable.

```rust
#[repr(align(128))]
struct PaddedAtomic(AtomicUsize);
```

In C++:

```cpp
alignas(std::hardware_destructive_interference_size) std::atomic<size_t> tail;
alignas(std::hardware_destructive_interference_size) std::atomic<size_t> head;
```

`hardware_destructive_interference_size` is 128 on most modern x86 builds.

## Per-language padding idioms

Every fast queue in every language pads contended fields. The exact mechanism differs:

| Language | Idiom |
|----------|-------|
| C++ | `alignas(std::hardware_destructive_interference_size)` (128 on most modern x86 builds), or `alignas(64)` for older code |
| Rust | `#[repr(align(128))]`, or `crossbeam_utils::CachePadded<T>` |
| Java | `@jdk.internal.vm.annotation.Contended` (JDK 8+, originally `@Contended` in `sun.misc`) |
| Go | manual `_ [128]byte` padding fields between contended atomics |
| C | `alignas(64)` from C11, or compiler-specific `__attribute__((aligned(128)))` |

The languages that get this right by default are vanishingly few. Most lock-free libraries in every language ship a `CachePadded<T>` wrapper because the language's primitives don't ensure padding automatically.

## Empirical impact

From the [[the-fastest-queue|fastest queue]] benchmarks comparing identical SPSC ring buffers with different padding:

- Unpadded (head/tail in same line): baseline
- 64-byte padded (one cache line): ~1.8× over baseline
- 128-byte padded (defeats adjacent prefetcher): **~3.1× over baseline**

The 64→128 step alone is worth roughly 1.7×, which is larger than most algorithmic improvements.

## Where this matters most

- **[[spsc-queue|SPSC ring buffers]]** — head and tail indices must be 128 bytes apart. This single padding decision is the largest perf knob in SPSC design.
- **[[lmax-disruptor|LMAX Disruptor]]** — explicitly pads sequence counters to defeat false sharing; the design's signature.
- **[[lcrq|LCRQ]]** — per-slot padding contributes to its ~400 MB memory consumption under load. [[scq|SCQ]] cuts this by half by relaxing per-slot padding while preserving padding on the contended head/tail.
- **Worker thread metadata** — per-CPU stats, allocator caches ([[tcmalloc]], [[jemalloc]]), [[thread-per-core]] runtime queues — anywhere two threads touch nominally separate fields.

## Beyond padding: shadow variables

Padding stops *spatial* sharing. Cache coherence costs from *temporal* sharing — both threads legitimately reading each other's values — are reduced via **shadow variables**: each thread keeps a local cached copy of the remote thread's index, checks the local copy first, and only does the cross-core read when the cached value indicates the queue may be full or empty. This converts per-operation cache-line transfers into per-batch transfers, which is the source of the 20× gap between naive and optimized [[spsc-queue|SPSC]] implementations.

## See also

- [[cache-coherency]] — the underlying MESI protocol cost
- [[lmax-disruptor]] — mechanical sympathy gold standard for padding
- [[ring-buffer]] — the structure most affected by false sharing
- [[spsc-queue]] — where padding and shadow variables compound
- [[memory-ordering]] — the orthogonal-but-related ordering question
- [[inter-thread-communication]] — broader latency hierarchy
