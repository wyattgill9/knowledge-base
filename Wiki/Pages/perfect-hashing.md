---
tags:
  - performance
  - data-structures
  - architecture
sources:
  - "Raw/Fastest CS/The fastest hash map in computer science, 2025.md"
last_updated: 2026-04-29
---

# Perfect Hashing

A class of hash table constructions that guarantee **zero collisions** for a known, static key set — making lookups branch-free and probe-free. For read-only data, perfect hashing is unbeatable; nothing in the [[swiss-table]] family can match it because a Swiss Table still has to probe at least once. The price is paid at construction time, which can be slow.

## The basic idea

Given a fixed set S of keys known up front, build a hash function h such that h is injective on S — every key maps to a distinct slot, no probing needed. Lookups become: hash, index, single key compare. The construction explores hash function families until it finds one with no collisions on S, sometimes via a two-level scheme (FKS hashing) or via perfect-hash builders that compose multiple hash functions.

Modern variants:

- **PTHash** — minimal perfect hashing with very compact representation; query time competitive with Swiss Tables once the function is built.
- **RecSplit** — recursive splitting; some of the smallest published representations (~1.56 bits per key, near the information-theoretic floor).
- **`fph::DynamicFphMap`** — supports dynamic insertions while retaining perfect-hash query speed when stable; rebuilds on growth.

## When it wins

- **Read-only static dictionaries** — compiled-in lookup tables, language identifiers, fixed enum mappings, geo lookups.
- **Read-mostly hot paths** where the construction cost amortizes over millions of queries.
- **Compact storage** — RecSplit-class structures store close to log₂(N!) bits, far less than even [[folly-f14]].

## When it loses

- **Write-heavy workloads** — every insertion may force a rebuild.
- **Unknown key set** — the construction needs the full set.
- **Cold lookups** — for one-shot queries the construction time dominates.

For dynamic workloads, use [[boost-unordered-flat-map]] or [[abseil-flat-hash-map]]. For static read-only data, perfect hashing is the right tool — and the speedup over even the best Swiss Table is real because there is no probing whatsoever.

## Related ideas

C#/.NET's [[frozen-dictionary]] takes an analogous approach without going all the way to perfect hashing: build-time analysis of the key set produces a read-optimized layout. Frozen dictionary is faster than mutable `Dictionary<TKey, TValue>` but does not guarantee the zero-collision property. See [[fastest-hash-map-2025]] for context.
