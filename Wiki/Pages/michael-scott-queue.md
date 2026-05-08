---
tags:
  - concurrency
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Michael-Scott Queue

The Michael-Scott queue (Maged Michael & Michael Scott, PODC 1996) is the canonical lock-free MPMC linked-list queue and was the production standard for nearly two decades. It is what Java's `ConcurrentLinkedQueue` implements. Modern designs ([[lcrq|LCRQ]], [[scq|SCQ]], [[lprq|LPRQ]]) outperform it by **3× or more** under contention, but it remains the reference implementation everyone learns first.

## How it works

The queue is a singly-linked list with separate `head` and `tail` pointers, each updated via CAS:

1. **Enqueue** — allocate a new node, CAS the current tail's `next` from null to the new node, then CAS the `tail` pointer forward. If the second CAS fails, another thread already advanced the tail, so cooperate by helping advance it on the next attempt.
2. **Dequeue** — read the head's next, return its value, CAS the `head` forward.

The "helping" pattern — when CAS fails because someone else made progress, finish their work — is the structural ancestor of [[wcq|wCQ]]'s wait-free helping mechanism, though wCQ uses it explicitly only on the slow path.

## Why it loses to LCRQ

The Michael-Scott queue's core problem is the [[faa-vs-cas|CAS retry loop on a contended pointer]]. Under N threads:

- Every push targets the same tail pointer.
- Only one CAS succeeds per round; the others retry.
- Throughput is O(log N) per round, so it goes flat or *decreases* past ~32 threads.

[[lcrq|LCRQ]] replaces this with FAA on an integer slot index — every thread succeeds in O(1), throughput scales with thread count, and the per-slot CAS that follows is uncontended. The 3× gap reported in the LCRQ paper is the difference between contention-melting and contention-tolerant primitives.

## Other costs

- **Per-node allocation** — every push allocates a new node, costing 50–100 ns even with a fast allocator. LCRQ allocates per ring (4,096 elements per allocation amortized), not per element.
- **Cache locality** — nodes are scattered; dequeue stalls on cache misses. Ring-buffer queues are contiguous.
- **The ABA problem** — if a node is freed and reused with the same address before a thread's CAS completes, the CAS can succeed incorrectly. Standard mitigations include [[crossbeam-epoch|epoch-based reclamation]], hazard pointers, or version-tagged pointers (which is why some implementations need 128-bit CAS — the same constraint LCRQ would later weaponize).

## Where it still appears

- **Java** — `java.util.concurrent.ConcurrentLinkedQueue` is a Michael-Scott queue. Java has not yet adopted LCRQ-class designs in the standard library, though JCTools provides faster alternatives.
- **Educational use** — clear enough to teach the lock-free queue concept, which is why it remains the canonical example in concurrency textbooks.

For new code, prefer [[lcrq|LCRQ]] (x86) or [[scq|SCQ]]/[[lprq|LPRQ]] (portable). Even Java has JCTools' `MpmcArrayQueue` (~6.5× faster than `ConcurrentLinkedQueue`).

## The bigger pattern

The Michael-Scott queue is the [[harris-linked-list|Harris linked list]] of the queue world: textbook standard, structurally important, no longer the throughput leader. Both designs share the same retraversal-on-CAS-failure weakness; both were superseded by designs that either restructured the contention point ([[lcrq|LCRQ]]'s FAA-on-a-ring for queues, Träff-Pöter's `fetch_or` marking for lists) or abandoned the linked-list shape entirely. See [[fastest-linked-lists]] for the full lineage of "how linked-shaped concurrent structures got faster by becoming less linked-list-shaped."

## See also

- [[lcrq]] — the FAA-on-a-ring design that obsoleted Michael-Scott for performance
- [[faa-vs-cas]] — why CAS-retry is the wrong primitive for contention hotspots
- [[mpmc-queue]] — modern landscape
- [[the-fastest-queue]] — full hierarchy
- [[harris-linked-list]] — sibling design for ordered sets, with the same retraversal weakness
