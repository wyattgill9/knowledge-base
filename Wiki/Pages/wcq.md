---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# wCQ (Wait-free Circular Queue)

wCQ (Nikolaev & Ravindran, **SPAA 2022**) is the wait-free MPMC queue that overturned the conventional wisdom that wait-freedom requires giving up throughput. It performs **on par with the best lock-free designs** by adopting a fast-path-slow-path methodology: [[scq|SCQ]] runs the fast path, and only when a thread fails after `MAX_PATIENCE` attempts does it switch to a cooperative slow path where all threads assist the stuck operation. In practice the slow path triggers so rarely that overhead is negligible.

## What "wait-free" means and why it used to be expensive

Lock-free guarantees that *some* thread makes progress; wait-free guarantees that *every* thread completes its operation in a bounded number of steps. The latter rules out starvation, but historically it cost throughput — early universal-construction wait-free transformations added 5–10× overhead, and direct wait-free queue designs (e.g., Kogan-Petrank 2011) were significantly slower than lock-free competitors.

The fundamental tradeoff is that wait-freedom requires a *helping mechanism*: when a fast thread sees a slow thread's pending operation, it has to complete it on the slow thread's behalf. That helping logic is overhead in every operation, even when no one needs help.

## Fast-path-slow-path

wCQ's insight is to run the wait-free helping logic only when actually needed. The state machine is:

1. **Fast path** — execute the SCQ algorithm normally (lock-free, no helping).
2. **Patience counter** — if a single operation fails MAX_PATIENCE times in succession, escalate.
3. **Slow path** — publish the operation in a per-thread descriptor and have other threads cooperatively help complete it.

Under realistic workloads the fast path succeeds essentially always — contention bursts that would starve a thread are rare and brief. The slow path is engineered to be slow but bounded; the fast path is engineered to be exactly as fast as SCQ.

## Performance

Published numbers on the same MPMC benchmarks used to evaluate LCRQ and SCQ:

- **Matches SCQ throughput** within measurement noise across thread counts
- **Bounded per-operation completion time** — the wait-free guarantee
- Outperforms previous wait-free queues (Kogan-Petrank, Yang-Mellor-Crummey) by orders of magnitude

## What this changes

The lock-free vs wait-free distinction has historically been treated as a hard practical tradeoff: choose throughput or progress guarantees. wCQ — together with Alistarh, Censor-Hillel, and Shavit's theoretical result that lock-free algorithms are "practically wait-free" — argues that for most applications, the distinction matters less than getting the cache behavior and atomic instruction choices right.

For hard real-time or safety-critical systems where bounded completion is required (avionics, audio DSP at 48 kHz, motor control), wCQ removes the historical reason to avoid wait-freedom. For everything else, [[lcrq|LCRQ]] + [[aggregating-funnels]] is still the throughput ceiling.

## See also

- [[scq]] — the lock-free queue that runs wCQ's fast path
- [[lcrq]] — the throughput champion (lock-free)
- [[the-fastest-queue]] — full hierarchy
- [[faa-vs-cas]] — the underlying primitive choice
