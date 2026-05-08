---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Synchronization Primitives

Inter-thread synchronization mechanisms span six orders of magnitude in latency, and the right choice depends almost entirely on **how deeply the mechanism touches the OS**. Atomic spin sits at the cache-coherence floor (35–100 ns RTT). Modern futex-based mutexes add only ~25 ns over raw atomics in the uncontended case. Once a syscall is required, latency jumps to 1–5 µs. A full context switch costs 1.2–10 µs. A thread sleep/wake cycle reaches 5–50 µs and pollutes caches.

## The latency ladder

| Level | Mechanism | Typical latency | CPU cost |
|-------|-----------|-----------------|----------|
| **0: Hardware floor** | Cache-line transfer | 16–65 ns | Negligible |
| **1: Atomic spin** | [[busy-spin\|Busy-wait on atomic flag]] | 35–100 ns RTT | 100% of one core |
| **2: Userspace mutex** | Futex fast path (uncontended) | 25–50 ns | Minimal |
| **3: Syscall** | Futex slow path / eventfd | 1–5 µs | Low |
| **4: Context switch** | Condition variable wake | 1.2–10 µs | Moderate |
| **5: Scheduler** | Thread sleep/wake cycle | 5–50 µs | High (cache pollution) |

Each level represents a categorical change, not a tuning knob. The largest single jump is between Level 2 and Level 3 — going from staying in userspace to making a system call costs roughly 50× in latency. This is why every fast inter-thread mechanism is engineered to stay above Level 2.

## Why `std::mutex` is not the bottleneck you think

A common assumption is that mutexes are slow because they "block." Modern futex-based implementations on Linux (glibc, musl) and the equivalent on Windows (`SRWLOCK`) are **adaptive**: they spin briefly checking if the lock holder is running on another core, then fall back to `futex(FUTEX_WAIT)` only if necessary.

Travis Downs measured that under moderate contention (3 threads), only **~0.18 futex syscalls occur per mutex operation** — over 80% of operations complete entirely in userspace. The uncontended fast path is a single CAS on the lock word; the contended fast path adds a brief PAUSE-loop spin; only the truly congested case crosses into the kernel.

Measured costs on Skylake under 2-thread contention:

| Operation | Latency |
|-----------|---------|
| Atomic add (FAA) | ~50 ns/op |
| CAS-based add | ~65 ns/op |
| `std::mutex`-protected add | ~125 ns/op |

The mutex is roughly 2× the bare atomic — not the 100× penalty folklore suggests. The implication: **for short critical sections at low-to-moderate contention, `std::mutex` is fine**. The cases where it's not fine are: hot paths where every nanosecond matters (drop to lock-free), very high contention (where futex contention itself becomes the bottleneck), and real-time systems (where any potential syscall is unacceptable).

## eventfd is slower than UNIX sockets

A surprising data point: **`eventfd` (4,353 ns RTT) is slower than UNIX domain sockets (1,439 ns)** for thread signaling. Both are orders of magnitude slower than atomic-based approaches because every notification requires a syscall. The eventfd advantage is integrating into `epoll` for multi-fd notification, not raw signaling speed.

`io_uring` reduces this overhead for I/O via shared-memory submission and completion rings, but for pure inter-thread signaling, futex-based approaches remain faster.

## Adaptive mutex implementations

Glibc's `pthread_mutex` adaptive type (`PTHREAD_MUTEX_ADAPTIVE_NP`) and the default behavior of `std::mutex` on most modern stdlibs follow this pattern:

```
1. Try CAS to acquire (uncontended fast path)
2. On failure, check if holder is running (via per-thread state in glibc 2.35+)
3. If holder is running: PAUSE-spin briefly, retry CAS
4. If holder is not running OR spin budget exhausted: futex(FUTEX_WAIT)
5. On unlock: CAS to release; if waiters exist, futex(FUTEX_WAKE)
```

The futex syscall is the slow path. The fast path stays in userspace. This is why mutex performance is dominated by [[cache-coherency|cache coherence costs]] on the lock word, not syscall costs.

## When to drop below mutexes

The hierarchy of "fast enough":

- **`std::mutex` for short critical sections** — almost always fine.
- **`std::shared_mutex` (RW lock)** for read-heavy workloads — but [[rcu|RCU]] dominates this niche when readers vastly outnumber writers.
- **Atomic flags + [[busy-spin|busy-spin]]** when latency must be sub-100 ns and a full core can be dedicated.
- **Lock-free queues ([[spsc-queue|SPSC]] / [[lcrq|LCRQ]])** when throughput is paramount.
- **[[flat-combining|Flat combining]]** when contention is so high that lock-free CAS retries cause meltdown.

## Cross-platform notes

| Platform | Userspace primitive | Slow-path mechanism |
|----------|---------------------|---------------------|
| Linux | atomic CAS on lock word | `futex(FUTEX_WAIT/WAKE)` |
| macOS | atomic CAS on lock word | `__ulock_wait/wake` (was `psync_*`) |
| Windows | atomic CAS on `SRWLOCK` | `WaitOnAddress` / `NtWaitForKeyedEvent` |
| FreeBSD | atomic CAS | `umtx` |

The interfaces differ but the cost model is the same: ~25 ns userspace fast path, ~1–5 µs slow path. Pthreads' `pthread_cond_wait` always involves a syscall on wake, putting it in Level 4–5; this is why notification-heavy code paths (producer signals consumer that data is available) are dominated by condvar latency unless replaced with [[busy-spin|spin-waiting]] or futex-based custom protocols.

## See also

- [[busy-spin]] — staying at Level 1 with `PAUSE` instructions
- [[cache-coherency]] — what sets Level 0
- [[faa-vs-cas]] — primitive choice within Level 1
- [[inter-thread-communication]] — broader hierarchy
- [[flat-combining]] — alternative under extreme contention
