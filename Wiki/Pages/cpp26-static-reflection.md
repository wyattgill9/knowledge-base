---
tags:
  - cpp
  - design-patterns
  - type-theory
  - architecture
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# C++26 Static Reflection

C++26 static reflection is the long-awaited feature that lets C++ code ask the compiler about the structure of types — what members a struct has, what enumerators an enum has, what the parameters of a function are — and generate code from the answer. It introduces the **`^T`** operator (reflexpr) which yields a `std::meta::info` handle for type `T`, and **splicers** (`[: ... :]`) which inject reflected entities back into code. Together they retire the macro-and-codegen layer that has defined C++ serialization, ORM mapping, RPC bindings, and reflection-shaped pattern matching for thirty years.

## The pattern it ends

For every serialization library in C++ history — boost::serialization, protobuf-generated code, cereal, nlohmann::json's macro-based ADL hooks — the same problem appeared: there is no way in standard C++ to enumerate the fields of a struct. Workarounds came in three flavors:

1. **Preprocessor macros**: declare members through `BOOST_FUSION_ADAPT_STRUCT(...)` or similar. Ugly, fragile, debugger-hostile.
2. **External codegen**: write a `.proto` or `.fbs` file, run a generator to emit C++ classes plus serialization code. Builds slow down; the source of truth moves out of C++.
3. **Custom DSLs / reflection libraries**: Boost.Hana, Boost.PFR, magic_get — heroic template metaprogramming that worked for aggregates but broke on edge cases.

Static reflection is the language-level answer:

```cpp
template <typename T>
std::string to_json(const T& obj) {
    std::string result = "{";
    bool first = true;

    template for (constexpr auto member :
                  std::meta::nonstatic_data_members_of(^T)) {
        if (!first) result += ",";
        first = false;
        result += std::format(R"("{}":{})",
            std::meta::identifier_of(member),
            to_json_value(obj.[:member:]));
    }
    return result + "}";
}

struct Point { double x; double y; double z; };
auto json = to_json(Point{1.0, 2.0, 3.0});
// → {"x":1.0,"y":2.0,"z":3.0}
```

That entire function — generic over any aggregate struct, emitting JSON whose keys match field names — is the standard library's compile-time mechanism doing the work. No macro registration, no external generator, no template-metaprogramming wizardry. The `^T` produces a reflection handle; `nonstatic_data_members_of` returns a `std::vector<std::meta::info>` at compile time; the `template for` (also new in C++26) iterates a compile-time sequence; the splicer `[:member:]` inserts the field access.

## The primitives

| Operation | Purpose |
|---|---|
| `^T` | Reflect a type, value, namespace, or template — yields `std::meta::info` |
| `[: x :]` | Splice — inject the reflected entity into the current context |
| `std::meta::nonstatic_data_members_of(refl)` | Enumerate fields |
| `std::meta::enumerators_of(refl)` | Enumerate enum values |
| `std::meta::identifier_of(refl)` | Get the name as a `std::string_view` |
| `std::meta::type_of(refl)` | Get the type |
| `template for (...)` | Compile-time loop over a sequence of reflections |

All operations are `consteval` — they execute purely at compile time and produce no runtime work in the emitted code.

## Use cases that change

**Serialization.** JSON, BSON, MessagePack, YAML — every library can generate per-type encoders/decoders from the struct definition. The Rust analog is `#[derive(Serialize, Deserialize)]` via `serde_derive`; static reflection gives C++ the same ergonomics without the macro layer. See [[serde-architecture]] for the design that C++ libraries will likely converge toward.

**ORM and database mapping.** `to_sql(record)` becomes generic over any struct. Column-name mismatches become compile errors.

**CLI / config parsing.** `from_args<Config>(argc, argv)` reflects on `Config`'s fields and builds the parser automatically.

**RPC / IPC.** Service definitions become C++ structs; client stubs are generated from reflection rather than from `.proto` files.

**Test scaffolding.** Property-based testing tools (the C++ analog of [[proptest]]) can generate arbitrary instances of any struct by reflecting on its fields.

**Compile-time validation.** Static asserts that walk a struct and check invariants — "no field is wider than 8 bytes," "all string fields are `std::string` not `char*`" — become trivial.

## Where it doesn't reach

Static reflection is read-only. You cannot synthesize new types from reflected data (no "create a struct with these fields at compile time"). C++26 reflection deliberately stops short of full metaclasses (the more ambitious Herb Sutter proposal from 2017), which would have allowed declarative class generation. A second wave of proposals — provisionally targeting C++29 — is expected to add **token injection** (generate code at compile time from a string-like representation) and a metaclass mechanism on top of the reflection foundation.

For now, the read-only reflection covers ~90% of the use cases that previously demanded macros or codegen, which is the practical win.

## Compile time and tooling

Reflection-heavy code increases compile time — every reflected operation runs the constant evaluator. Early measurements on Clang's experimental implementation suggest 1.5–3× compilation overhead for code that heavily uses `template for` over reflected member lists, though the constant evaluator is improving rapidly. The debug/IDE story is also evolving: at the time of writing, navigation across splicers is supported in clangd ≥ 20 but spotty elsewhere.

## Why this is the C++26 headline feature

The argument for static reflection being the most consequential C++26 feature is that it removes a *category* of tooling rather than refining individual ones. Every C++ codebase has a serialization layer; most have several, generated by different tools. Static reflection lets a small library replace the lot — and lets new domains (column store mappers, GPU buffer layouts, GUI introspection) get the same treatment without needing new codegen pipelines. See [[modern-cpp-design-patterns]] for the broader convergence with Rust's `derive` macro ecosystem and Swift's `Mirror`.

The Rust comparison: Rust's `#[derive(...)]` macros run on the AST and emit code, but they have no access to type information across compilation units. C++26 reflection gives full access to types within a translation unit and propagates through templates — strictly more powerful in that respect, though Rust's procedural macros are more flexible in others (they can call arbitrary Rust code at compile time, including I/O). The two systems are different points on the same design axis.
