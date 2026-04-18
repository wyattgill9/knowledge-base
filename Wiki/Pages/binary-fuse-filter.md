---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Binary Fuse Filter

The best all-around probabilistic set membership filter, surpassing Bloom filters in both space and speed. Created by Thomas Mueller Graf and Daniel Lemire (2022), Binary Fuse filters achieve within **13% of the information-theoretic space minimum** (vs 44% for Bloom), with **2x faster construction** than Xor filters and comparable 3-memory-access query time.

## The filter landscape

| Filter | Space overhead | Best for | Deletion? |
|--------|---------------|----------|-----------|
| Binary Fuse | 13% | Static sets (default choice) | No |
| Blocked Bloom | 44% | Raw query throughput (single cache-line + AVX2) | No |
| Ribbon (Meta) | ~0.5% (BuRR) | LSM trees, write-once/read-many | No |
| Cuckoo (CMU) | Variable | Dynamic sets needing deletion | Yes |
| Xor | 23% | Middle ground | No |
| Classic Bloom | 44% | Legacy, simplicity | No (or counting) |

## Why Bloom filters lost

Classic Bloom filters use k independent hash functions mapping to k bit positions. At typical false-positive rates, they waste ~44% more space than the information-theoretic minimum. Binary Fuse filters use a different construction: XOR-based fingerprint storage across three hash positions, achieving near-optimal space with the same 3-memory-access query pattern.

## Production use

Binary Fuse filters are used in production by databend, ntop, and InfiniFlow. **Ribbon filters** (Peter Dillinger, Meta) are integrated into RocksDB since v6.15, offering ~7 bits/key for 1% false-positive rate vs ~10 for Bloom — a significant win for LSM tree filter blocks.

## Decision framework

Use **Binary Fuse** for static sets — it is the default choice. Use **Cuckoo filters** when you need deletion support. Use **Ribbon filters** in LSM-tree contexts (RocksDB, LevelDB-style). Use **blocked Bloom** when raw query throughput is the only metric that matters.
