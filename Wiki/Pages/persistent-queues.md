---
tags:
  - data-structures
  - type-theory
  - performance
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# Persistent Queues

A persistent (functional, immutable) queue preserves all previous versions when modified — every push and pop returns a new queue, with the old one still usable. Naively, this would cost O(n) per operation. The genuinely surprising result is that **worst-case O(1)** persistent queues exist, through increasingly sophisticated rebuilding techniques. The trade-off versus a simple [[ring-buffer]] is constant-factor overhead from incremental rebuilding — usually 5–20× slower in practice — which buys persistence and (in the most ambitious designs) catenability.

## The naive amortized version

Two stacks: one for the front, one (reversed) for the back. Push to the back stack; pop from the front stack; when the front empties, reverse the back into a new front. Amortized O(1) per operation. **Not** worst-case O(1) — the reversal is O(n) when it happens, which destroys worst-case latency guarantees and breaks under persistence (because the same expensive reversal can be triggered repeatedly on different versions).

## Hood-Melville (1981)

The first persistent queue with worst-case O(1). Uses **global rebuilding**: when the back stack reaches half the size of the front, it begins reversing it incrementally — a constant amount of work per subsequent operation — so the reversal completes before the next one is needed. Complex to implement but provably worst-case constant.

## Okasaki (1995/1998)

Chris Okasaki simplified Hood-Melville with a **scheduling** technique: lazy evaluation handles the reversal as a suspended computation, and an explicit schedule of "forced thunks" ensures each suspended computation is fully evaluated before its result is needed. Maintains a list of pending suspensions and forces one per operation. The result has the same asymptotic complexity but significantly cleaner code, and became the standard presentation in Okasaki's *Purely Functional Data Structures* (1998).

## Kaplan-Tarjan (1999)

The most ambitious persistent queue result: a **real-time deque with O(1) catenation**. Supports:

- `push_front`, `pop_front`, `push_back`, `pop_back` — all worst-case O(1)
- `concat(d1, d2)` — append two deques in worst-case O(1)

Catenation is the killer feature — most queue / deque designs that achieve O(1) push/pop do *not* support efficient append. The Kaplan-Tarjan deque uses a hierarchy of buffers and "yellow/red/green" balance invariants to maintain O(1) bounds even under arbitrary catenation patterns.

It is so complex that a **verified OCaml implementation was only achieved in 2025**, more than 25 years after the paper. For most uses, simpler amortized-O(1) persistent deques are preferred — but the Kaplan-Tarjan deque is the answer to "is it possible at all" for hard real-time persistence.

## When to use which

- **Imperative code** — use a [[ring-buffer]]. Worst-case O(1) is trivial; persistence is not needed.
- **Functional code, no real-time requirement** — use the simple two-stack amortized version (or whatever's in your language's standard library — `Data.Sequence` in Haskell uses finger trees with O(log min(i, n-i)) random access and amortized O(1) at the ends).
- **Functional code with real-time requirement** — Okasaki's real-time queue. Hood-Melville if you need to avoid laziness. Kaplan-Tarjan only if you need O(1) catenation.

For hard real-time systems (audio DSP at 48 kHz gives ~21 μs per sample, motor control similarly tight), the higher constant factors of worst-case O(1) functional queues are worth paying because amortized O(n) blowups are unacceptable.

## See also

- [[ring-buffer]] — the imperative answer
- [[the-fastest-queue]] — full hierarchy
- [[recursion-schemes]] — adjacent functional-data-structure machinery
