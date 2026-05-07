---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Elimination-Backoff Stack

The Elimination-Backoff Stack (Hendler, Shavit & Yerushalmi, SPAA 2004) is the canonical fast lock-free LIFO. It dramatically outscales the classic Treiber stack at high thread counts by **matching complementary push/pop operations in a randomized elimination array**, bypassing the central head-pointer bottleneck. At 64 threads with a 50/50 push/pop workload it reaches **100,000+ ops/msec**.

## The Treiber bottleneck

A Treiber stack (1986) is the simplest lock-free stack: a singly-linked list with a CAS on the head pointer for both push and pop. Under contention, every operation contends on that single pointer — the same [[faa-vs-cas|CAS retry storm]] that limits the [[michael-scott-queue]]. Treiber doesn't degrade as catastrophically as Michael-Scott (push and pop don't both target the tail), but throughput still flattens past ~16 threads.

## The elimination insight

A push and a pop on a stack *cancel each other out* — semantically equivalent to no operation at all on the underlying structure. If you can match a pending push with a pending pop before either reaches the head pointer, neither needs to touch shared state.

The Elimination-Backoff Stack uses a randomized **elimination array**: when a thread's head-pointer CAS fails, it backs off to a random slot in the array and waits briefly. If a complementary operation lands in the same slot during the wait, they exchange values and both return without touching the stack. Otherwise the thread retries the head-pointer operation.

Under the heavy contention where Treiber stacks degrade, elimination opportunities are abundant — the array becomes the *primary* path, with the head pointer as a fallback. Throughput scales near-linearly with thread count rather than flattening.

## Where it sits in the queue/stack landscape

Stacks are simpler than queues — the head is the only contention point, and pushes/pops are symmetrical. This means stack-specific tricks (elimination) are available that don't apply to queues. The closest queue analog is [[aggregating-funnels]], which uses a different mechanism (FAA combining) for a similar goal (avoiding contention on a single hot location).

## In practice

- **Java `java.util.concurrent.ConcurrentLinkedDeque`** — uses a variant of elimination internally
- **liblfds** and **Concurrency Kit** — provide elimination-backoff stack implementations in C
- **Rust** — no widely-used native implementation; the `crossbeam-utils::Backoff` primitive is the canonical "wait briefly under contention" tool but doesn't include the elimination array

For most Rust use cases, a simple Treiber stack suffices because contention is rarely high enough to matter. When it is, the right answer is usually to redesign for [[thread-per-core|thread-local]] state plus aggregation rather than fight contention on a shared stack.

## See also

- [[concurrent-queues]]
- [[faa-vs-cas]] — why the Treiber CAS bottleneck exists
- [[aggregating-funnels]] — the analogous queue-side technique
- [[the-fastest-queue]]
