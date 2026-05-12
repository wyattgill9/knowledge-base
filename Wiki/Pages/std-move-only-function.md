---
tags:
  - cpp
  - design-patterns
  - type-theory
sources:
  - "Raw/C++/Modern C++ Design Patterns (C++23 and Beyond).md"
last_updated: 2026-05-11
---

# `std::move_only_function`

`std::move_only_function` is C++23's type-erased callable wrapper that, unlike its older sibling `std::function`, does not require its stored callable to be copyable. It accepts any move-only callable — closures capturing `std::unique_ptr`, captured `std::move_only_function` chains, FFI handles — and enables clean value-semantic polymorphism without inheritance hierarchies. It is the C++ analog of Rust's `Box<dyn FnOnce>` (and, with the proposed `std::function_ref` in C++26, of `&dyn Fn`).

## The deficiency of `std::function`

`std::function<R(Args...)>` (C++11) requires the stored callable to be `CopyConstructible`. That rules out a large fraction of modern lambdas — anything capturing a `std::unique_ptr`, a `std::thread`, a `std::promise`, a `std::move_only_function`, or any pre-existing move-only type. For fifteen years C++ developers worked around this with `std::shared_ptr` captures, custom type erasure, or `std::packaged_task`, all of which add overhead or boilerplate.

```cpp
// std::function — won't compile, lambda captures unique_ptr
std::function<void()> f = [p = std::make_unique<Resource>()]() { p->use(); };

// std::move_only_function — compiles, no allocation overhead
std::move_only_function<void()> g = [p = std::make_unique<Resource>()]() { p->use(); };
```

## The polymorphism pattern

The wider point is that `std::move_only_function` enables clean value-semantic polymorphism — closing the gap between "I want runtime dispatch" and "I want value semantics" that previously forced developers into class hierarchies:

```cpp
struct RenderPipeline {
    using Stage = std::move_only_function<void(FrameBuffer&) const>;

    void add_stage(Stage stage) { stages_.push_back(std::move(stage)); }
    void execute(FrameBuffer& fb) const {
        for (auto& stage : stages_) stage(fb);
    }

private:
    std::vector<Stage> stages_;
};

pipeline.add_stage([shader = load_shader("pbr")](FrameBuffer& fb) {
    shader.apply(fb);
});
pipeline.add_stage(ToneMappingPass{2.2});  // function object, also fine
```

No `class IStage { virtual void apply(...) = 0; }`. No factory functions returning `std::unique_ptr<IStage>`. No virtual destructor mistakes. Each stage is a value, lives in the `std::vector` by value, and dispatches via the type-erased call operator. Add a new stage and you write a new lambda, not a new class.

## The C++26 companion: `function_ref`

`std::function_ref<R(Args...)>` (proposed for C++26, already shipping in `tl::function_ref`) is the non-owning counterpart — it refers to a callable without owning it, like a callable `std::span`. The decision matrix:

| You want... | Use |
|---|---|
| Owning, copyable callable | `std::function` |
| Owning, move-only callable | `std::move_only_function` |
| Non-owning reference to a callable | `std::function_ref` (C++26) |

`function_ref` is the right choice for parameters of higher-order functions — passing a comparison callback to a sort, a visitor to a traversal — because it avoids the small-buffer-or-heap allocation that `function` and `move_only_function` may do.

## The cost model

Both `std::function` and `std::move_only_function` are typically implemented with Small Buffer Optimization — a small inline storage area (commonly 16–24 bytes) plus a function pointer for the indirect call. Captures that fit go inline; larger captures heap-allocate. The call site is a single indirect function-pointer dispatch, comparable in cost to a virtual call but without the `vptr` indirection on the object side.

For hot paths where the callable type is known statically, a template parameter is faster — inlining beats type erasure. `move_only_function` is the right tool when you genuinely need runtime polymorphism: pipelines, plugin systems, command queues, event handlers.

## The pattern shift

This is one of the clearest cases in [[modern-cpp-design-patterns]] of value semantics displacing inheritance. The Gang-of-Four "Strategy" pattern was a class hierarchy with virtual methods; the modern strategy is a `std::move_only_function` member. The GoF "Command" pattern was an `ICommand` base with `execute()`; the modern command is a stored callable. The GoF "Observer" pattern was a list of `IObserver*`; the modern observer list is `std::vector<std::move_only_function<void(Event)>>`.

The pattern names survive in textbooks; the implementations look nothing like them.
