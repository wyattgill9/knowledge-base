---
tags:
  - performance
  - data-structures
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# FrozenDictionary

C#/.NET 8's read-optimized immutable dictionary, taking a different angle on the "fastest hash map" question: analyze the keys at construction time and pick a lookup strategy specialized to that key set. The result is **43–69% faster string lookups** than mutable `Dictionary<TKey, TValue>` — about 4.3 ns versus 10–18 ns for small dictionaries.

## The construction-time analysis

When you call `dict.ToFrozenDictionary()`, the runtime inspects the keys and chooses among several internal layouts: integer-key tables, length-bucketed string tables, ordinal-comparison string tables, single-substring-hash tables, and small-array linear search for tiny dictionaries. The chosen layout is then specialized — different code paths for different key shapes.

This is conceptually the dual of [[swiss-table]]: instead of a single architecture that handles all key types well, FrozenDictionary picks the *best* architecture for *this* key set. The cost is amortized at build time and frozen forever after.

## When it pays off

The break-even point is roughly **5,000–630,000 reads** depending on dictionary size. Below that, the construction cost outweighs the per-lookup savings. Above that, FrozenDictionary wins decisively.

Right choice for:
- Configuration loaded at startup and queried for the process lifetime.
- Compiled lookup tables (HTTP headers, MIME types, error code maps).
- Routing tables, feature flag maps, anything built once and read forever.

Wrong choice for:
- Anything mutable.
- Dictionaries that turn over frequently.
- Cold dictionaries hit only a handful of times.

## How it relates to perfect hashing

FrozenDictionary stops short of full [[perfect-hashing]] — it does not guarantee zero collisions. It picks a *good* layout per key set, not an optimal one. The construction is much faster than perfect-hash builders (PTHash, RecSplit), and the lookup cost is correspondingly slightly higher. For most application workloads it is the right pragmatic point on the curve: significant speedup, fast enough to build at startup, no special tooling.

## Other languages

- C++: similar role played by `frozen` (Quentin Quintet), a constexpr perfect-hash library that builds at compile time.
- Rust: `phf` for compile-time perfect hashing of static maps. `OnceLock<HashMap>` for runtime-built read-only maps without specialized layout.
- Java: no equivalent; primitive-specialized maps (Koloboke, fastutil) provide their own speedup axis.

See [[fastest-hash-map-2025]] for context and [[perfect-hashing]] for the upper-bound version of this idea.
