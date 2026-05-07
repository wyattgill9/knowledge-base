---
tags:
  - rust
  - concurrency
  - performance
  - crate
sources:
  - "Raw/Fastest CS/The fastest queue in all of computer science.md"
last_updated: 2026-05-06
---

# rtrb

`rtrb` (Real-Time Ring Buffer) is the leading wait-free [[spsc-queue|SPSC]] queue for Rust, designed for hard real-time use cases like audio DSP and motor control. Push and pop are both wait-free with **~7 ns per operation** on x86 and benchmarks reaching **520–530M ops/s** on Apple M4 — placing it at the top of the [[the-fastest-queue|published SPSC tier]] regardless of language.

## Design

A bounded power-of-two [[ring-buffer]] with [[false-sharing|128-byte padding]] between the producer and consumer indices, acquire/release atomics rather than seq_cst, and **shadow variables** (the producer caches the consumer's head, and vice versa) to convert per-operation cache-line transfers into per-batch transfers. Standard SPSC technique applied without compromise.

The API is split: constructing the queue returns a `(Producer, Consumer)` pair, and `Send` is implemented for each side but `Sync` is not. The split-handle design enforces the SPSC contract at the type level — there is no way to accidentally push from two threads.

## Comparison

| Crate | Throughput | Notes |
|-------|-----------|-------|
| **rtrb** | ~7 ns/op, 520M+ ops/s on M4 | Wait-free, hard real-time |
| `bounded-spsc-queue` | Comparable | Similar approach, less actively maintained |
| `ringbuffer-spsc` | 520–530M ops/s on M4 | The 530M number above |
| [[crossbeam-array-queue\|crossbeam ArrayQueue]] | ~10× slower | MPMC, not SPSC |

For MPMC use [[crossbeam-array-queue|crossbeam ArrayQueue]] or [[kanal]]; for message-passing channels [[crossbeam-channel]] or [[kanal]]. For raw SPSC throughput, rtrb.

## When to use it

- Audio and DSP pipelines (the original motivation)
- Telemetry / log buffers between exactly one producer and one consumer
- [[thread-per-core]] mailbox between adjacent shards
- Anywhere you can guarantee single-producer / single-consumer topology

## See also

- [[spsc-queue]] — the optimization techniques rtrb applies
- [[ring-buffer]] — the underlying structure
- [[the-fastest-queue]] — full hierarchy
