---
tags:
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Huge Pages

Huge pages are 2 MB or 1 GB virtual memory mappings backed by a single TLB entry, replacing the standard 4 KB page that Linux uses by default. For [[inter-thread-communication|inter-thread communication]] and database workloads, huge pages **eliminate TLB misses across hot data**. A single 2 MB huge page covers 32× more address space than a standard 4 KB page; a 1 GB huge page covers 16,384× more. For a multi-megabyte ring buffer or a multi-gigabyte hash table, the TLB hit rate goes from ~50% to ~100%, removing one of the largest hidden costs in cache-friendly algorithms.

## Why TLB misses matter

The TLB (Translation Lookaside Buffer) caches virtual-to-physical address translations. A modern x86 CPU has roughly:

- 64 entries in L1 dTLB for 4 KB pages → covers 256 KB
- 32 entries in L1 dTLB for 2 MB pages → covers 64 MB
- 1024–2048 entries in L2 TLB

A TLB miss costs 10–100 cycles for a page-table walk. For a workload that touches more memory than the TLB can cover, every access can pay this cost — even when the data itself is in L1 cache. This is the silent killer of "cache-friendly" data structures: the data fits, but the page tables don't.

A 2 MB huge page replaces 512 standard pages with a single TLB entry, raising TLB-covered memory by 32×. A 1 GB huge page replaces 524,288 standard pages, covering an entire working set in one entry.

## Configuration on Linux

Two mechanisms exist:

**Transparent Huge Pages (THP)** — the kernel automatically promotes contiguous 4 KB pages to 2 MB huge pages when possible.

```sh
echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

THP is convenient but has costs: page promotion is asynchronous and can cause latency spikes when `khugepaged` runs, and fragmentation reduces the success rate over time. Many database vendors ([[parlayhash]] / [[swiss-table]] heavy workloads, MongoDB, Redis) recommend disabling THP and using explicit huge pages instead.

**Explicit huge pages** — reserved at boot or runtime and allocated via `mmap` with `MAP_HUGETLB`:

```sh
# Boot args for 1 GB pages (must be at boot, can't be allocated later)
default_hugepagesz=1G hugepagesz=1G hugepages=8

# 2 MB pages can be reserved at runtime
echo 1024 > /proc/sys/vm/nr_hugepages
```

```c
void *buf = mmap(NULL, 2 * 1024 * 1024,
                 PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                 -1, 0);
```

For 1 GB huge pages specifically, allocation must happen at kernel boot — physical fragmentation makes it unlikely to succeed at runtime on a long-running system.

## Where huge pages matter most

- **[[inter-thread-communication|Shared ring buffers]]** between cores. A 4 MB SPSC ring buffer needs only 2 huge-page TLB entries instead of 1024 standard ones; once both producer and consumer cores have those entries cached, all subsequent address translations are zero-cost.
- **Hash tables** larger than ~1 MB ([[parlayhash]], [[abseil-flat-hash-map]], [[boost-unordered-flat-map]]) benefit dramatically — random probing is exactly the access pattern that thrashes TLBs.
- **Large arrays / matrix operations** in HPC and ML workloads.
- **Database working sets** — buffer pools, B-tree internal nodes, log structured trees.

## Where huge pages are neutral or harmful

- **Small data** (< 128 KB) — fits in standard TLB entries already; huge pages waste memory if allocated unnecessarily.
- **Memory-pressured systems** — huge pages cannot be swapped out as easily, so they reduce flexibility under pressure.
- **Random allocation patterns** — many small heap allocations don't benefit from huge pages and can suffer from worse fragmentation under THP.
- **NUMA misalignment** — a 1 GB huge page must come from a single NUMA node, so a poorly-pinned thread can end up with all of its hot data on the wrong socket.

## Allocator integration

Some allocators integrate huge-page support directly:

- **[[jemalloc]]** — has `metadata_thp` and `thp` options; can be configured to back arenas with huge pages.
- **[[mimalloc]]** — supports huge-page allocation via `MIMALLOC_RESERVE_HUGE_OS_PAGES`.
- **[[tcmalloc]]** — uses huge pages internally for its central heap on supported systems.

Switching to a huge-page-aware allocator gives the benefit without source changes, which is a significant ergonomic win compared to manual `mmap(MAP_HUGETLB)`.

## HFT impact

In the [[core-pinning|HFT optimization stack]], huge pages contribute meaningfully to P99.9 tail latency:

- TLB miss rate on hot ring buffers drops from ~5% to ~0%.
- Page table walks during cache-line refills disappear, removing a class of latency outliers.
- Combined with [[core-pinning|core pinning]] and [[busy-spin|busy-spin synchronization]], huge pages are part of the recipe that drives sub-microsecond P99.9.

The effect on median latency is small (a few ns); the effect on tail latency is large (microseconds removed from outliers).

## See also

- [[core-pinning]] — pinned threads + huge pages is the HFT recipe
- [[inter-thread-communication]] — where huge pages plug into the broader stack
- [[jemalloc]] / [[mimalloc]] — allocators with huge-page support
- [[direct-io]] — companion technique for storage
