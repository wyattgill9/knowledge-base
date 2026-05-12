---
tags:
  - cpp
  - design-patterns
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# Deducing `this`

Deducing `this` is C++23's *explicit object parameter* feature — written `this auto&& self` — which makes the implicit `this` pointer a named, deducible parameter of a member function. It quietly retires the [[crtp|Curiously Recurring Template Pattern]], unlocks recursive lambdas without `std::function`, and lets a single member function definition cover the const/non-const, lvalue/rvalue, and base-vs-derived axes that previously required four overloads or a CRTP cast.

## The CRTP it replaces

For two decades, the standard way to inject a method into derived classes without paying for virtual dispatch was CRTP — a base class templated on the derived type, casting `this` to `Derived*` at every call:

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
```

This works, but the base class becomes a template (cascading into every signature that references it), the `static_cast` is unchecked, and value-category propagation through the base is fragile. Deducing `this` collapses the whole construct:

```cpp
struct Base {
    void interface(this auto&& self) {
        self.implementation();
    }
};

struct Widget : Base {
    void implementation() { /* ... */ }
};
```

`self` deduces to `Widget&`, `const Widget&`, or `Widget&&` depending on the call site. The base class is no longer a template. There is no cast. Value categories propagate through `auto&&` correctly. This is the cleanest single feature in C++23 measured by lines of code removed from real codebases — Microsoft's STL implementation alone deleted hundreds of CRTP scaffolding lines on adoption.

## Recursive lambdas, trivially

Before C++23, a recursive lambda required either a `std::function` wrapper (heap allocation, type erasure) or a Y-combinator–style trampoline. With deducing `this`, the lambda receives itself as its first parameter:

```cpp
auto factorial = [](this auto&& self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
```

No heap allocation, no `std::function`, no auxiliary types. The compiler generates the unique closure type and `self` refers to it directly.

## The collapsing-overload-set win

Deducing `this` also collapses the four-way overload sets that infested ref-qualified accessor methods:

```cpp
// Pre-C++23: four near-identical overloads
class Widget {
    Resource& get() &        { return r_; }
    const Resource& get() const & { return r_; }
    Resource&& get() &&      { return std::move(r_); }
    const Resource&& get() const && { return std::move(r_); }
};

// C++23
class Widget {
    auto&& get(this auto&& self) {
        return std::forward<decltype(self)>(self).r_;
    }
};
```

One definition, perfect forwarding of value category. This is the kind of refactor that quietly removes a class of subtle bugs (someone forgetting the `&&` overload and silently copying on rvalue access).

## Where it doesn't replace CRTP

Deducing `this` does *not* give you the static type of the derived class inside the base for cases that need it at compile time — e.g., a CRTP base whose data layout depends on `sizeof(Derived)`, or that uses `Derived::static_constant` in its own type definitions. For those cases CRTP is still the right tool. The replacement holds for the vastly more common "I want to call a derived method from a base method" case, which is almost always the actual use.

## Position in the pattern shift

This sits at the head of [[modern-cpp-design-patterns]] because it is the cleanest single example of the broader shift: a feature that retires an entire named pattern rather than refining it. The 2002–2022 C++ programmer learned CRTP as a rite of passage; the 2025+ programmer learns deducing `this` and may never need the older idiom outside legacy code.
