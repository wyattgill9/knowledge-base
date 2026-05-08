---
tags:
  - concurrency
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest ways to talk between threads.md"
last_updated: 2026-05-07
---

# Flat Combining

Flat combining (Hendler, Incze, Shavit, Tzafrir, SPAA 2010) takes a counterintuitive approach to concurrent data structure design: instead of parallelizing access, it **serializes operations through a single combiner thread** that batches them. Under very high contention, this outperforms lock-free queues because it eliminates [[faa-vs-cas|CAS retry storms]] entirely. The combiner amortizes the cache-coherence cost of acquiring the structure across many operations, paying once for what would otherwise be N coherence transactions.

## How it works

Each thread that wants to operate on the shared structure publishes its request to a thread-local slot in a shared "publication list." Then:

1. Try to acquire the structure's lock.
2. **If acquired (combiner role)**: scan the publication list, execute all pending operations from all threads, write results back to each thread's slot, release the lock.
3. **If not acquired (waiter role)**: spin briefly waiting for the result to appear in the local slot.

The combiner sees the full batch of pending work in one acquisition. Because the list is thread-local in publication but shared in scan, the cache traffic per operation is dominated by the combiner's single sequential walk — not N independent contended writes to a hot pointer.

## Why it wins under extreme contention

Lock-free designs based on CAS suffer from contention meltdown: as thread count rises, CAS failure rates rise, retries waste cache traffic, and total throughput **decreases** past some thread count. [[faa-vs-cas|FAA-based queues]] like [[lcrq|LCRQ]] mostly dodge this, but FAA itself bottlenecks on the cache-coherence round-trip rate.

Flat combining sidesteps both problems by ensuring that exactly one thread touches the contended cache lines per round of operations. The cost per operation drops from "one cache line bounce per thread" to "one cache line bounce per batch" — and the batch size grows automatically with contention, because more contending threads means more pending operations in the publication list.

The technique scales in the opposite direction from lock-free: **higher contention makes flat combining faster**, while it makes CAS-based designs slower.

## Where it shows up

- **Stacks and queues under extreme contention** — the original SPAA 2010 paper benchmarked LIFO stacks; later work extended to FIFO queues, priority queues, and skip lists.
- **[[aggregating-funnels|Aggregating Funnels]] (PPoPP 2025)** — tree-structured combining specifically applied to FAA, achieving up to 2.5× over plain LCRQ at high thread counts. This is flat combining's modern descendant.
- **Hardware transactional memory** is conceptually similar: a single transaction batches multiple operations on shared state, paying coherence cost once.

## Trade-offs

Flat combining's strength — single-threaded execution of a batch — is also its weakness. The combiner is a serialization point, so:

- **Single-thread throughput sets the ceiling.** No amount of additional threads can push beyond what the combiner alone can execute. Lock-free designs can in principle scale past this point if contention is low enough.
- **Latency is bursty.** A waiter's request might be at the front of a batch (low latency) or buried behind 50 others (high latency). For latency-sensitive systems, this is unacceptable; for throughput-sensitive systems, it's a win.
- **Combiner failure is catastrophic.** If the combiner is descheduled mid-batch, all waiters stall. Real implementations bound waiter spin time and re-elect a combiner after a timeout.

## When to use it

Use flat combining (or its descendants) when:

- Thread count vastly exceeds available cores (> 64 contending threads is where it starts to dominate).
- The data structure is updated on every operation — read-only structures should use [[rcu|RCU]] instead.
- Latency variance is acceptable; throughput is the goal.

For typical 4–16 thread workloads with mixed read/write patterns, lock-free designs ([[lcrq|LCRQ]], [[swiss-table|Swiss-Table]] variants, [[crossbeam-array-queue|Vyukov]]) are simpler and faster. Flat combining is a heavy weapon for the high-contention extreme.

## See also

- [[aggregating-funnels]] — modern combining for FAA-based queues
- [[faa-vs-cas]] — why CAS retry meltdown is the problem combining solves
- [[elimination-backoff-stack]] — different approach to contention (matching pairs)
- [[inter-thread-communication]] — where combining fits in the broader landscape
