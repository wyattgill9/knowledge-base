---
tags:
  - performance
  - architecture
  - concurrency
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Kernel Bypass

Kernel bypass moves I/O off the kernel's hot path, replacing syscall-based interfaces with direct userspace access to hardware queues. **DPDK** is the canonical example: NIC packet rings mapped into a userspace process, polled by dedicated cores in tight loops, with zero syscalls per packet. UDP latency drops from **~9 µs** through the kernel stack to **~5 µs** with DPDK. The same design pattern — dedicated polling cores, pre-allocated buffers, lock-free coordination — translates almost directly to inter-thread work.

## What "the kernel" costs in I/O

A standard `recv()` for a UDP packet involves: NIC interrupt → softirq handler → packet copy into kernel sk_buff → routing decision → socket buffer copy → wake target process → context switch into userspace → copy from kernel buffer to user buffer → return from syscall. Each step adds ~hundreds of ns. The total ~9 µs end-to-end UDP latency is dominated by these steps, not by the wire.

Kernel bypass collapses this to: NIC writes directly to a userspace ring → poll thread reads it. No interrupt, no copy, no context switch, no syscall. The remaining latency is the NIC's own queue + memory bandwidth + the application's processing.

## The DPDK pattern

The [Data Plane Development Kit](https://www.dpdk.org/) embodies the kernel-bypass pattern:

- **NIC ownership transfer**: a hardware NIC is unbound from the kernel driver and bound to DPDK's userspace driver (vfio-pci or uio).
- **Memory pools**: pre-allocated huge-page-backed packet buffers, never freed.
- **Poll-mode drivers (PMD)**: dedicated cores spin reading the NIC's RX ring.
- **Lockless ring buffers** for inter-core packet handoff (DPDK's `rte_ring` is essentially a [[mpmc-queue|Vyukov MPMC ring]]).
- **CPU pinning**: every PMD thread pinned via [[core-pinning|`taskset`]] to a specific core.
- **[[huge-pages|1 GB huge pages]]** for packet pools, eliminating TLB misses on packet headers.

The throughput numbers: 100M+ packets/sec on a 100 Gbps NIC, with sub-microsecond per-packet latency. The cost is dedicating cores to polling — DPDK PMDs run at 100% CPU even with no traffic.

## Why this matters for inter-thread work

Every primitive in the DPDK stack maps to inter-thread communication:

| DPDK technique | Inter-thread analogue |
|----------------|------------------------|
| Poll-mode driver | [[busy-spin\|Busy-spin consumer]] |
| `rte_ring` (lockless) | [[spsc-queue\|SPSC]] / [[lcrq\|LCRQ]] |
| Memory pool (pre-alloc) | Pre-allocated [[ring-buffer\|ring buffer]] entries |
| Hugepage backing | [[huge-pages\|Hugepage]]-backed shared memory |
| CPU pinning | [[core-pinning\|`taskset` + `isolcpus`]] |
| No syscalls | No mutexes, no condvars |

The high-frequency-trading playbook is essentially DPDK's principles applied to thread-to-thread instead of NIC-to-thread. Same mental model: predict where work happens, dedicate cores to it, eliminate every layer that wasn't strictly necessary.

## io_uring: kernel bypass without bypassing the kernel

[[io-uring]] is Linux's answer to kernel bypass for general-purpose I/O. It uses shared-memory submission and completion rings between userspace and the kernel — userspace writes submission queue entries, the kernel processes them, and completion entries appear in userspace without a syscall (or with one syscall amortized over a batch). For pure inter-thread *signaling*, this is overkill; for I/O that needs to coexist with kernel features (TCP, filesystems, device drivers), it's the modern recommendation.

The key distinction:

- **Full kernel bypass (DPDK)**: maximum performance, gives up kernel networking stack, no firewalls/iptables/netfilter, complex deployment.
- **io_uring**: gets most of the wins (batched submissions, no syscall per op, async by design), keeps kernel features, simpler deployment.

For inter-thread communication specifically, neither is the answer — futexes and atomic spinning beat both. Kernel bypass becomes relevant when threads talk *to hardware* (network, storage), not to each other.

## When kernel bypass is the right call

- **Network packet processing** at line rate (10 Gbps and above with non-trivial per-packet work).
- **Storage I/O** in HFT order processing where every microsecond from the wire to the matching engine matters.
- **GPU/accelerator interaction** where the syscall-per-submission model dominates throughput.
- **NVMe direct access** via SPDK (the storage analogue of DPDK).

When kernel bypass is *wrong*:

- General-purpose servers — the kernel's stack handles 99% of cases at 99% of the speed.
- Anything that needs to coexist with iptables, eBPF, conntrack, or other kernel features.
- Workloads where the cost of dedicating cores to polling exceeds the latency savings (most non-HFT workloads).

## See also

- [[io-uring]] — Linux's compromise between bypass and stack integration
- [[core-pinning]] — required for kernel-bypass to deliver
- [[huge-pages]] — required for buffer-heavy bypass workloads
- [[direct-io]] — storage-side analogue (O_DIRECT bypassing page cache)
- [[busy-spin]] — the consumer-side discipline
- [[inter-thread-communication]] — where bypass principles translate
