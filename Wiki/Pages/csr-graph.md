---
tags:
  - data-structures
  - performance
  - architecture
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Compressed Sparse Row (CSR)

The universally acknowledged fastest read-only graph representation. By storing all neighbor arrays contiguously in a single edge array with a vertex offset array, every graph access reduces to an indexed array lookup with maximal cache locality.

## Performance gaps are enormous

Framework benchmarks confirm massive differences: **graph-tool** (C++/Boost backend with OpenMP) and **NetworKit** (KIT) are **40–250x faster** than NetworkX for standard algorithms. NetworKit's PageRank on Pokec completes in 0.2 seconds versus 59.6 for igraph. The reason is simple: CSR is a flat array, and adjacency lists are pointer-chasing linked structures. The same principle that makes [[b-tree]] beat red-black trees and [[soa-vs-aos|SoA beat AoS]].

## The mutation problem

CSR's weakness is O(n+m) cost for any edge mutation — you have to rebuild the entire edge array. This has driven innovation in dynamic graph structures:

**Terrace** (Pandey et al., 2021) uses hierarchical storage (in-place arrays, packed memory arrays, B-trees by vertex degree) to achieve 2x faster than Aspen on graph algorithms while supporting fast batch insertions.

**CSR++** (Oracle Labs) stays within 10% of CSR's read performance while enabling 10x faster insertions — the best trade-off for mixed read/write workloads.

## GPU acceleration

For GPU-accelerated analytics, **Gunrock** (UC Davis) and **GraphBLAST** (Berkeley Lab) achieve order-of-magnitude speedups over CPU frameworks using frontier-based CUDA abstractions over CSR storage. The CSR layout maps naturally to GPU memory access patterns.

## Compressed representations

For web-scale graphs, the **WebGraph framework** (Boldi & Vigna) achieves 2–6 bits per link by exploiting power-law degree distributions, locality of reference, and the "copy property" of web graphs. **k2-trees** push this to 1.66–2.55 bits per link with the added advantage of supporting both forward and reverse neighbor queries.
