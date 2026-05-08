---
tags:
  - performance
  - concurrency
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Core Pinning

Pinning a thread to a specific physical core is the single highest-leverage optimization in any [[inter-thread-communication|inter-thread communication]] system — larger than any choice of data structure. The reason is that [[cache-coherency|cache-line transfer latency varies 5–10×]] depending on which cores are talking: 16 ns within an AMD Zen 3 CCX, 84–107 ns crossing the Infinity Fabric, 150 ns cross-socket on Intel. A scheduler that migrates a thread between cores invalidates all of its L1/L2 cache and resets the topology assumptions every queue and atomic was tuned for. The fix — `taskset`, `isolcpus`, and `SCHED_FIFO` — is the foundation of the high-frequency-trading playbook.

## What pinning does

Linux's CFS scheduler is free to migrate any thread to any core at any time, optimizing for fairness across the whole system. For latency-sensitive workloads, this is exactly wrong: every migration is a cold-cache restart costing **50–200 ns per evicted cache line** and resetting all branch predictor state. Core pinning takes the migration decision away from the kernel:

- **`taskset -c 4 ./app`** — restrict the process to CPU 4. Variants restrict to ranges (`-c 4-7`) or sets (`-c 4,8,12`).
- **`pthread_setaffinity_np(3)`** — per-thread pinning from inside the program. Used to pin worker N to core N in [[thread-per-core|thread-per-core runtimes]].
- **`sched_setaffinity(2)`** — the underlying syscall. Equivalent to `taskset` but settable per-thread at runtime.

## isolcpus: removing cores from the scheduler

`taskset` only constrains *your* threads — the kernel can still schedule arbitrary other work on those cores. To make pinned cores truly exclusive, boot the kernel with `isolcpus=4-7` (or the modern `cpuset` cgroup equivalent) which removes those cores from the general scheduler entirely. Combined with `nohz_full=4-7` (skip timer ticks on isolated cores) and `rcu_nocbs=4-7` (offload RCU callbacks), the result is cores that run *only* the threads you pin to them, with minimal kernel interference.

The measured impact: HFT benchmarks report tail latency dropping from **120 µs to 30 µs at the 99.9th percentile** — a 4× improvement at the worst case — purely from `isolcpus` + `taskset`. Median latency barely moves; this optimization buys you tail-latency stability, not average speed.

## SCHED_FIFO: preventing preemption

Even on isolated cores, kernel threads (kworker, ksoftirqd, RCU callbacks) can occasionally preempt your application. `SCHED_FIFO` is a real-time scheduling class that preempts everything below it:

```c
struct sched_param param = { .sched_priority = 50 };
pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
```

Set with care: an infinite-loop `SCHED_FIFO` thread with no yield will lock up the system, because nothing can preempt it. The kernel partial-mitigates this with `kernel.sched_rt_runtime_us` (default: 950000 µs out of every 1000000 µs reserved for RT, leaving 5% for non-RT work) — set to `-1` to disable the safety net entirely if you trust your code.

## Topology awareness

The naive "pin to core 0 and core 1" approach can be the worst possible choice if those cores are in different clusters. The same SPSC ring buffer that hits 16 ns RTT between two cores in the same Zen 3 CCX hits 107 ns between cores in different chiplets — same code, same data, **6.7× slower**.

The right approach: query topology and pin deliberately.

```sh
# Show CPU topology
lscpu --extended
# Show cache hierarchy
lscpu --caches
# Show NUMA layout
numactl --hardware
```

For SPSC queues, pin the producer and consumer to cores that share an L3 cache. For [[thread-per-core]] runtimes with per-shard queues, the within-CCX/within-socket constraint lets you partition shards by NUMA node and run cross-shard messaging only when topology cooperates. See [[numa-aware-queues]].

## Disabling SMT (hyperthreading)

Two SMT siblings share execution units, L1 cache, and the entire core's resources. Two threads on the same physical core via SMT can signal each other in **16.5 ns** (faster than cross-core), but they also contend for every functional unit. For latency-sensitive workloads, the contention dominates the proximity benefit, and disabling SMT is the standard HFT recipe.

```sh
echo off > /sys/devices/system/cpu/smt/control
```

This halves logical core count but makes the remaining cores run with predictable execution characteristics. For throughput-oriented workloads (web servers, databases), keep SMT on — the contention rarely matters and the extra logical threads improve utilization.

## The full HFT recipe

```sh
# Boot args
isolcpus=4-7 nohz_full=4-7 rcu_nocbs=4-7 intel_pstate=disable

# At runtime
echo off > /sys/devices/system/cpu/smt/control
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
echo -1 > /proc/sys/kernel/sched_rt_runtime_us

# Run
taskset -c 4-7 chrt -f 50 ./app  # SCHED_FIFO priority 50, cores 4–7
```

Combined with [[huge-pages|1 GB huge pages]] and [[busy-spin|busy-spin synchronization]], the result is sub-microsecond P99.9 latency for the inner loop of an HFT trading engine. None of this is needed for a typical web service — the cost is roughly 4× overprovisioning (cores 4–7 cannot run anything else) and dramatically harder operational story.

## When to pin

- **Always pin** for [[thread-per-core]] runtimes — that's the entire model.
- **Pin for latency-sensitive** queues, especially [[spsc-queue|SPSC]] with [[busy-spin|busy-spin]] consumers.
- **Pin for benchmarks** even if production won't pin — otherwise migration noise dominates measurements.
- **Don't pin** general server workloads where utilization matters more than tail latency. CFS does fine for HTTP servers, batch jobs, and most application code.

## See also

- [[thread-per-core]] — server architecture built on pinning
- [[cache-coherency]] — why topology matters
- [[numa-aware-queues]] — cross-socket considerations
- [[busy-spin]] — what to do once pinned
- [[inter-thread-communication]] — where pinning fits
