---
tags:
  - cpp
  - design-patterns
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Curiously Recurring Template Pattern (CRTP)

CRTP is the C++ idiom of templating a base class on its own derived type — `struct Widget : Base<Widget>` — so the base can call methods on the concrete derived type without virtual dispatch. James Coplien named it in 1995, and for nearly three decades it was the standard way to inject behavior into a derived class with zero runtime overhead, used for everything from mixin patterns to expression templates to static polymorphism. C++23's [[deducing-this]] retires the vast majority of CRTP uses; the pattern is now mostly archaeological for new code, though it remains the right tool for the narrow case where the base class genuinely needs the derived type at compile time (sizeof-dependent layouts, static-constant access).

## The original mechanic

```cpp
template <typename Derived>
struct Base {
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

struct Widget : Base<Widget> {
    void implementation() { /* ... */ }
};

Widget w;
w.interface();  // statically dispatches to Widget::implementation
```

The base class is a template; the derived class is the instantiation. Inside the base, `static_cast<Derived*>(this)` is the unchecked cast that gives the base access to the derived type's methods. Zero virtual functions, zero vtable, full inlining — that was the win.

## The uses CRTP enabled

- **Static polymorphism mixins** — the most common use, now subsumed by [[deducing-this]].
- **Expression templates** — Eigen, Blaze, xtensor build their lazy linear-algebra DSLs on CRTP. The base type encodes the expression structure; the derived types are the leaves. This use is still common because the base needs the derived type's dimensions and value type at compile time.
- **Counted instances / registry patterns** — `Base<Derived>::count` gives each derived class its own static counter.
- **Fluent interfaces with covariant return types** — every method returns `Derived&` so chaining preserves the static type.

## Why deducing `this` displaces most uses

The single biggest cost of CRTP was that the base class became a template. Every signature that referenced it cascaded into a template. `Base<Widget>` and `Base<Thing>` are unrelated types. You couldn't put them in a container, couldn't pass them to a non-templated function, couldn't write a polymorphic API over them without re-erasing through a non-CRTP interface.

[[deducing-this]] eliminates the cascade. The base is not a template; only the *member function* is templated on `this auto&& self`. You can have a non-template `Base` whose `interface()` method deduces its own static type from the call site. For the simple "I want to call derived methods from the base" case, deducing `this` is strictly better.

## Where CRTP still wins

CRTP remains the right tool when:

1. **The base class layout depends on the derived type.** `template <typename D> struct Base { D::value_type stored; };` — `sizeof(Base<D>)` depends on `D`. Deducing `this` cannot help; the base type itself needs to vary.
2. **The base needs derived static constants for its own declarations.** `template <typename D> struct Base { std::array<int, D::N> buffer; };` requires `D::N` at the type level of `Base`.
3. **Expression-template DSLs.** Eigen-style lazy evaluation pipelines need to know the leaf-node types when composing — the entire structure of `MatMul<Add<A, B>, C>` is in the type. Deducing `this` doesn't reach here.
4. **You need to dispatch on the base type in templates external to the hierarchy.** A `template<typename D> void process(Base<D>& b)` can't be expressed with a non-template `Base`.

For these cases — perhaps 5–10% of historical CRTP uses — the pattern remains correct and the displacement story does not apply.

## Position in modern C++

In [[modern-cpp-design-patterns]] CRTP appears as the canonical "pattern displaced by a language feature" example. Its history is a useful case study: a workaround so elegant that it became idiomatic and was taught for thirty years, only for the language to eventually add the feature CRTP was approximating (`this auto&&`) and retire the workaround. This is the standard cycle for successful language workarounds — CRTP joins `auto_ptr` (displaced by `unique_ptr`), function-pointer-driven callbacks (displaced by lambdas and [[std-move-only-function]]), and `enable_if` SFINAE (displaced by concepts) in the genealogy of C++ idioms that the language eventually absorbed.

The lesson, perhaps, is that durable C++ design happens at two layers: the workaround layer (which is fluent in current C++ and ages out over time) and the principle layer (zero-cost abstraction, value semantics, static polymorphism) which is stable across decades. CRTP was a workaround; the underlying principle — static polymorphism without virtual dispatch — survives in [[deducing-this]] unchanged.
