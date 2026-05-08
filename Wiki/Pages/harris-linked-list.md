---
tags:
  - concurrency
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Harris Linked List

Timothy Harris's **2001** lock-free ordered linked list. The marked-pointer logical-deletion technique it introduced — stealing a low bit of the `next` pointer to mark a node as logically deleted before unlinking it physically — remains the structural basis for nearly every modern concurrent linked list. It is the [[michael-scott-queue|Michael-Scott queue]] of the ordered-set world: textbook standard, historically influential, no longer the throughput leader.

## The two-step deletion protocol

A naive lock-free delete that simply CAS-unlinks a node races with concurrent inserts after that node — an inserter may add a successor to a node that's already being unlinked, losing the inserted element. Harris's solution:

1. **Logically delete.** CAS the deleted node's `next` pointer to a *marked* version (low bit set). Concurrent operations seeing the mark know the node is dead.
2. **Physically unlink.** A second CAS swings the predecessor's `next` past the marked node.

The mark bit is read atomically with the pointer it protects, which is why the protocol works on architectures where pointers are aligned (the low bits are normally zero). This is the same pointer-stealing trick later used in CAS2-based queues — Harris was first.

## The retraversal weakness

Harris's protocol pays a crippling cost under contention: a failed CAS forces **retraversal from the list head**. On a list of length L, contended deletes can cost O(L) work per attempt. For long lists at high thread counts the throughput collapses — exactly the [[faa-vs-cas|CAS-retry meltdown]] that obsoleted the [[michael-scott-queue]] for queue workloads.

## Träff-Pöter (2020)

Träff and Pöter addressed the retraversal problem with two techniques:

1. **Approximate backward pointers.** A failed CAS can resume near where it left off rather than restarting from the head. The "approximate" qualifier means the backward pointers can lag without compromising correctness — they're a hint, not an invariant.
2. **`fetch_or`-based marking.** Replace the marking CAS with an atomic OR, which never fails — analogous to the [[faa-vs-cas|FAA-over-CAS]] insight in queues. The first thread to set the bit wins; concurrent attempts simply observe it set.

Reported result: **orders-of-magnitude improvement** over textbook Harris in worst-case contention scenarios, and meaningful gains on random mixed workloads. Reference implementations are at `github.com/parlab-tuwien/lockfree-linked-list`. This is the current state of the art for **lock-free ordered linked lists**.

## VBL: provably concurrency-optimal

The **VBL algorithm** (Aksenov et al., PaCT 2021) is *concurrency-optimal* — it rejects only those concurrent schedules that would violate linearizability, accepting every schedule that could possibly be linearizable. **1.6× throughput** over Lazy Linked Lists at 72 threads. This is the strongest correctness-equivalence theorem known for any concurrent ordered linked list.

## Wait-free variants

**Timnat, Braginsky, Kogan, and Petrank (2012)** added wait-freedom to ordered-set linked lists with only **~2% overhead** versus lock-free counterparts via a fast-path/slow-path methodology — the wait-free helping mechanism triggers in fewer than 1 in 3,000 operations. The same fast-path/slow-path pattern reappears in [[wcq|wCQ]] for queues; the trick is general.

## Where it sits

Harris-Träff-Pöter and VBL are the leading designs for **lock-free ordered concurrent linked lists**. For unordered queue access, [[lcrq|LCRQ]] / [[scq|SCQ]] / [[wcq|wCQ]] dominate by abandoning the linked-list shape entirely in favor of FAA-on-a-ring. For ordered concurrent *maps* (rather than just linked lists), the modern winners are [[art-olc|ART-OLC]], [[masstree]], and [[bp-tree|BP-Tree]] — see [[fastest-ordered-maps]].

The recurring lesson across all these designs: **avoid CAS retries on contended pointers**. Harris's marked-pointer protocol works; its head-retraversal cost is the part subsequent designs (Träff-Pöter, VBL, [[lcrq|LCRQ]]) systematically eliminate. The structural insight has aged well; the original implementation has not.
