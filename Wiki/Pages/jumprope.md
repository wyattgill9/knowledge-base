---
tags:
  - rust
  - data-structures
  - performance
  - crate
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# JumpRope

The "world's fastest rope implementation" by Joseph Gentle (Rust). Achieves ~35–40 million edits/second on real editing traces — 3x faster than Ropey. Combines a skip-list spine with gap-buffer leaves, optimizing for the editing patterns that actually occur in text editors (localized insertions and deletions with occasional large jumps).
