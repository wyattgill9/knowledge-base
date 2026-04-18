---
tags:
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# simdjson

The demonstration of what [[simd-programming|SIMD]]-optimized data structure design can achieve at its ceiling. Created by Geoff Langdale and Daniel Lemire, simdjson parses JSON at gigabytes/second on a single core — **4x faster than RapidJSON, 25x faster than JSON for Modern C++**.

## How it works

simdjson processes 32–64 bytes per iteration using AVX2. Key techniques:

- **Carry-less multiplication** for string mask computation — identifying quoted regions without branch-heavy character-by-character scanning.
- **Structural character classification** via SIMD — finding `{`, `}`, `[`, `]`, `:`, `,` across 64 bytes simultaneously.
- **~50% fewer total instructions** than scalar alternatives, because each instruction processes 32–64 bytes instead of 1.

The parser produces a tape-like output (a flat array of tokens) rather than a tree, avoiding allocation overhead during parsing.

## Significance

simdjson is not just a fast JSON parser — it is proof that SIMD transforms data processing fundamentally. The same principles (batch classification, branchless scanning, flat output formats) apply to CSV parsing, log processing, XML parsing, and any structured text format. Daniel Lemire (co-author) is also behind [[binary-fuse-filter|Binary Fuse filters]], continuing the theme of applying hardware-aware design to fundamental data structures.
