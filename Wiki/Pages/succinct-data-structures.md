---
tags:
  - data-structures
  - performance
sources:
  - "Raw/Fastest CS/General.md"
last_updated: 2026-04-17
---

# Succinct Data Structures

Data structures that use space close to the information-theoretic minimum while still supporting efficient queries — trading CPU cycles for extraordinary space efficiency.

## SDSL-lite

The **SDSL-lite** library (Simon Gog et al.) is the reference implementation, packaging highlights of 40 research publications in C++ templates:

- **Bit vectors with rank/select** — answer "how many 1s before position i?" (rank) and "where is the j-th 1?" (select) in ~25–50 ns, using only o(n) bits beyond the raw bit vector.
- **Wavelet trees** — represent sequences over arbitrary alphabets, supporting rank/select/access in O(log σ) time with near-optimal space.
- **FM-indexes** — compressed full-text indexes based on the Burrows-Wheeler transform, handling inputs up to 64 GB.
- **Compressed suffix arrays** — full suffix array functionality in space proportional to the compressed text.

## When to use

Succinct structures excel when space is the primary constraint and you can afford moderate CPU overhead per query. Typical applications: bioinformatics (genome indexes), information retrieval (compressed inverted indexes), and any domain where the dataset is too large for conventional structures but random access is still needed.

## Relationship to other approaches

[[learned-indexes]] also achieve space reduction but through data-dependent modeling rather than information-theoretic compression. [[binary-fuse-filter|Binary Fuse filters]] apply similar space-optimality thinking to probabilistic membership queries. [[fst-finite-state-transducer|Finite State Transducers]] are succinct representations specifically for sorted string sets, sharing both prefixes and suffixes.
