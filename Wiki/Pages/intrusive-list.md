---
tags:
  - data-structures
  - performance
  - cpp
  - architecture
sources:
  - "Raw/Fastest CS/The fastest linked lists ever built.md"
last_updated: 2026-05-08
---

# Intrusive Linked List

A linked list where the prev/next pointers live **inside the data structures being linked**, not in separate node objects. The list does not own its elements; insertion, deletion, and traversal require zero heap allocation. **Boost.Intrusive** benchmarks show **5–29× faster combined insertion and destruction** versus `std::list`, because intrusive nodes share cache lines with their data and require no allocator round-trips.

## The Linux kernel's list.h

Arguably the most battle-tested linked list in existence. A circular doubly-linked intrusive list using a single `struct list_head` embedded directly in any kernel structure that wants to be listable:

```c
struct task_struct {
    /* ... lots of fields ... */
    struct list_head tasks;
    /* ... more fields ... */
};

LIST_HEAD(all_tasks);
list_add(&new_task->tasks, &all_tasks);
```

Three advantages compound:

1. **Cache locality with the data.** Following `tasks->next` lands on a cache line that already contains the surrounding `task_struct` fields the kernel was about to touch anyway.
2. **Zero allocation.** No `kmalloc` per list-add, no allocator lock to contend, no fragmentation.
3. **Multi-list membership for free.** A single struct can hold N `list_head` fields, joining N different lists simultaneously without duplication. Kernel objects routinely sit in dozens of lists this way.

Linus Torvalds removed `prefetch()` calls from the list traversal macros in **Linux 2.6.40** because the hardware prefetcher was already doing better than the software hints — and prefetching the NULL at list ends caused TLB misses. A useful cautionary tale: hand-rolled prefetch on linked structures usually loses to the branch predictor and the L2 streamer in modern CPUs. The 2025 [[fastest-linked-lists|Linkey]] hardware-software prefetcher is the inverse argument: prefetching for linked structures *can* work, but only with compiler-provided structural hints.

## Boost.Intrusive

The C++ generalization. `boost::intrusive::list` lets a single object participate in multiple lists by embedding multiple `list_member_hook` fields. Trade-off: callers manage element lifetime explicitly — the list cannot delete what it does not own. For systems where allocation discipline is already enforced (kernels, game engines, embedded), this is a *feature*, not a cost.

## Where it sits

In the [[fastest-linked-lists|hierarchy]], intrusive lists are the **systems-programming reference**. For higher-level C++ general use, [[plf-list|plf::list]] is the drop-in `std::list` replacement leader. For raw traversal, [[vector-backed-list|vector-backed lists]] with compaction are faster. For concurrent access, [[harris-linked-list|Harris-Träff-Pöter]] for ordered sets and [[lcrq|LCRQ]]/[[wcq|wCQ]] for queues. The intrusive model is unmatched specifically when allocation cost or multi-list membership dominates.

The cache-line sharing argument behind intrusive lists is the same one that powers [[ecs-pattern|ECS]] storage and [[soa-vs-aos|structure-of-arrays]] layouts — putting structurally related fields where the access pattern actually wants them.
