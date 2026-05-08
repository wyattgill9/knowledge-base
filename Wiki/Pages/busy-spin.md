---
tags:
  - concurrency
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Busy-Spin

Busy-spinning — repeatedly checking an atomic flag in a tight loop instead of sleeping — is the lowest-latency way for one thread to wait on another, achieving **35–100 ns RTT** versus 1–10 µs for any syscall-based notification. The cost is 100% of one core, permanently. The technique is mandatory at the latency-sensitive end of [[inter-thread-communication]] (HFT, audio DSP, kernel data planes), forbidden in general-purpose code (drains battery, starves other work), and surprisingly architecture-dependent: **`PAUSE` cost varies 15× across Intel generations**, from ~9 cycles on Broadwell to ~140 cycles on Skylake.

## The PAUSE instruction

A naive spin loop is a busy `while (!flag) ;`. This runs as fast as the CPU can execute, which has two problems on x86:

1. **Memory ordering speculation**: the CPU may reorder loads aggressively in a tight loop, then have to flush the pipeline when the speculation conflicts with the eventual store from the other thread — a 50+ cycle penalty per conflict.
2. **SMT fairness**: a tight loop on one logical core starves the other SMT sibling of execution-unit cycles.

The `PAUSE` instruction (mnemonic: `rep nop`) tells the CPU "I'm in a spin loop." The CPU responds by:

- Slowing down the loop (preventing memory ordering speculation conflicts).
- Yielding execution-unit cycles to the SMT sibling.
- Reducing power consumption modestly.

The semantics are unchanged — `PAUSE` is functionally a no-op — but the timing changes meaningfully.

## The 15× generational variance

`PAUSE` cost is **not stable across Intel generations**:

| Microarchitecture | PAUSE cost |
|-------------------|------------|
| Pre-Skylake (Sandy Bridge → Broadwell) | ~9 cycles |
| Skylake and later (Skylake-X, Ice Lake, etc.) | ~140 cycles |
| AMD Zen / Zen 2 / Zen 3 | ~50–65 cycles |

Intel deliberately changed Skylake's `PAUSE` to be much longer to improve SMT fairness — the original 9-cycle version was too short to give meaningful execution time to the SMT sibling. The result: a spin loop calibrated for Broadwell runs **15× slower** on Skylake (in terms of PAUSE-loop iterations per microsecond), which can break tuning assumptions for things like backoff loops and adaptive-spin mutex implementations.

The lesson: **don't hardcode PAUSE counts**. Either spin for a fixed time (using `rdtsc` to measure) or use a calibration pass at startup.

## ARM and Apple Silicon

ARM's equivalent is `YIELD`. Cost is much lower than x86's PAUSE because ARM's pipeline has different reorder properties — typically a few cycles on most cores. For long waits, **`WFE` (Wait For Event)** is more efficient: it puts the core into a low-power state until either an event signal (`SEV`) arrives or a timeout fires. The companion to `WFE` is `WFE`-aware spin loops where one thread armed with the data executes `SEV` to wake the waiter, avoiding both polling overhead and the OS scheduler.

Apple Silicon explicitly recommends `WFE` for spin loops in performance-critical code; `YIELD` works but isn't ideal.

## The portable answer

C++ provides `std::this_thread::yield()` (which is a syscall — Level 4 on the [[synchronization-primitives|cost ladder]], not what you want) and as of C++20 has `std::atomic_flag::wait()` for futex-style blocking. For pure busy-spin hints, use:

```cpp
#include <emmintrin.h>  // _mm_pause on x86
_mm_pause();
```

Or in Rust:

```rust
std::hint::spin_loop();  // emits PAUSE on x86, YIELD on ARM
```

The standard library hints emit the right instruction per target. Tuning the *number* of spins between checks remains per-platform.

## The exponential backoff pattern

A naive PAUSE-loop wastes power and SMT cycles when the wait is long. The standard pattern combines spinning with eventual sleeping:

```c
int spin = 16;
while (!ready_flag.load(std::memory_order_acquire)) {
    if (spin > 0) {
        for (int i = 0; i < spin; i++) _mm_pause();
        spin *= 2;
        if (spin > 1024) spin = 1024;  // cap
    } else {
        // long wait: drop to futex
        futex_wait(&ready_flag, 0);
    }
}
```

Short waits stay in the spin tier (Level 1, ~50 ns); long waits drop to the futex tier (Level 3, ~5 µs). This gives the best of both: fast common case, no power drain on rare long waits. Modern adaptive mutexes (glibc `pthread_mutex`, [[synchronization-primitives|Linux futex implementation]]) use this pattern internally.

## When to busy-spin

- **Hard real-time** — audio DSP, motor control, HFT order matching. Any potential syscall is unacceptable.
- **Dedicated polling cores** — DPDK packet processing, [[kernel-bypass|kernel-bypass]] designs, [[thread-per-core]] runtimes that own their cores.
- **Sub-microsecond critical sections** — where the cost of even a futex fast-path call (25–50 ns) is meaningful.

When *not* to busy-spin:

- General-purpose code on shared servers — drains power, hurts other tenants.
- Battery-powered devices.
- Containers without [[core-pinning|pinned cores]] — the busy-spinning thread can be preempted, defeating the entire point.
- Wait times longer than ~10 µs — the futex slow path becomes cheaper than continued spinning.

## See also

- [[synchronization-primitives]] — where busy-spin sits in the latency ladder
- [[memory-ordering]] — `PAUSE` and memory-ordering speculation interaction
- [[core-pinning]] — required for busy-spin to be effective
- [[spsc-queue]] — the data structure most often paired with busy-spin
- [[lmax-disruptor]] — explicitly uses busy-spin wait strategies
