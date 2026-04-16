A guide to design patterns leveraging the latest C++ standards — moving beyond classical GoF patterns into idiomatic, expressive, and zero-cost abstractions.

---

## 1. Deducing `this` — CRTP Killer

C++23's _explicit object parameters_ (`this auto&&`) eliminate the need for the Curiously Recurring Template Pattern entirely.

**Classic CRTP (pre-C++23):**

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

**Modern — Deducing `this`:**

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

No templates on the base class, no `static_cast`, full support for value categories. This also enables **recursive lambdas** trivially:

```cpp
auto factorial = [](this auto&& self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
```

---

## 2. `std::expected` — Monadic Error Handling

C++23's `std::expected<T, E>` replaces the old "return error code or throw" dilemma with a composable, Railway-Oriented Programming style.

```cpp
enum class ParseError { InvalidFormat, OutOfRange };

std::expected<int, ParseError> parse_int(std::string_view sv);
std::expected<double, ParseError> to_celsius(int fahrenheit);

auto result = parse_int(input)
    .and_then(to_celsius)
    .transform([](double c) { return std::format("{:.1f}°C", c); })
    .or_else([](ParseError e) -> std::expected<std::string, ParseError> {
        log_error(e);
        return std::unexpected(e);
    });
```

**Pattern:** Chain `.and_then()`, `.transform()`, and `.or_else()` to build pipelines that short-circuit on errors — no exceptions, no manual `if` checks.

---

## 3. `std::generator` — Lazy Sequence Pattern

C++23 coroutines with `std::generator<T>` make lazy, pull-based iteration a first-class pattern.

```cpp
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// Compose with ranges
auto first_even_fibs = fibonacci()
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::take(10);
```

This replaces hand-rolled iterator classes, callback-based generators, and complex state machines with simple, readable, sequential code.

---

## 4. `std::mdspan` — Multidimensional View Pattern

C++23's `std::mdspan` provides a zero-overhead, non-owning multidimensional view over contiguous data — replacing raw pointer arithmetic and custom matrix classes.

```cpp
void matrix_multiply(std::mdspan<const float, std::dextents<size_t, 2>> A,
                     std::mdspan<const float, std::dextents<size_t, 2>> B,
                     std::mdspan<float, std::dextents<size_t, 2>>       C) {
    for (size_t i = 0; i < C.extent(0); ++i)
        for (size_t j = 0; j < C.extent(1); ++j) {
            C[i, j] = 0;
            for (size_t k = 0; k < A.extent(1); ++k)
                C[i, j] += A[i, k] * B[k, j];
        }
}

// Usage — same flat buffer, different views
std::vector<float> buf(rows * cols);
auto matrix = std::mdspan(buf.data(), rows, cols);
```

Custom layout policies (`std::layout_left`, `std::layout_right`, or your own) let you change memory layout without changing algorithms.

---

## 5. Compile-Time Strategy with `constexpr` + `if consteval`

C++23's `if consteval` lets you select strategies at compile time vs. runtime within a single function body.

```cpp
constexpr double fast_sqrt(double x) {
    if consteval {
        // Compile-time: use Newton's method (no std::sqrt in constexpr before)
        double guess = x / 2.0;
        for (int i = 0; i < 20; ++i)
            guess = (guess + x / guess) / 2.0;
        return guess;
    } else {
        // Runtime: use hardware instruction
        return std::sqrt(x);
    }
}

constexpr double compile_time_val = fast_sqrt(2.0);  // Newton's method
double runtime_val = fast_sqrt(some_input);           // std::sqrt
```

---

## 6. Type-Erased Polymorphism with `std::move_only_function`

C++23's `std::move_only_function` (and the upcoming `std::function_ref` in C++26) enable clean, value-semantic polymorphism without inheritance hierarchies.

```cpp
struct RenderPipeline {
    using Stage = std::move_only_function<void(FrameBuffer&) const>;

    void add_stage(Stage stage) {
        stages_.push_back(std::move(stage));
    }

    void execute(FrameBuffer& fb) const {
        for (auto& stage : stages_)
            stage(fb);
    }

private:
    std::vector<Stage> stages_;
};

// Mix lambdas, function objects, captures — no base class needed
pipeline.add_stage([shader = load_shader("pbr")](FrameBuffer& fb) {
    shader.apply(fb);
});
pipeline.add_stage(ToneMappingPass{gamma: 2.2});
```

---

## 7. Static Reflection Pattern (C++26 — `^T` and Splicers)

C++26 introduces static reflection, enabling patterns that previously required macros or external code generators.

```cpp
// Auto-generate serialization from struct definition
template <typename T>
std::string to_json(const T& obj) {
    std::string result = "{";
    bool first = true;

    template for (constexpr auto member : std::meta::nonstatic_data_members_of(^T)) {
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

This replaces macro-based serialization, manual `to_json()` methods, and external code generators like protobuf for simple cases.

---

## 8. Sender/Receiver — Structured Concurrency (C++26 `std::execution`)

C++26's `std::execution` (formerly P2300) introduces a composable, structured concurrency model that replaces ad-hoc thread pools and callback chains.

```cpp
namespace ex = std::execution;

auto work = ex::when_all(
    ex::on(thread_pool.get_scheduler(),
        ex::just(url1) | ex::then(fetch) | ex::then(parse)),
    ex::on(thread_pool.get_scheduler(),
        ex::just(url2) | ex::then(fetch) | ex::then(parse))
)
| ex::then([](auto result1, auto result2) {
    return merge(result1, result2);
});

auto [merged] = ex::sync_wait(work).value();
```

**Key properties:** structured lifetimes (no dangling callbacks), cancellation propagation, composable pipelines, scheduler-agnostic algorithms.

---

## 9. Pattern Matching via `std::visit` + Overload Set (C++23 Enhanced)

While full pattern matching (`inspect`) is still proposed, the `overloaded` idiom with `std::visit` is now cleaner with C++23 deduction improvements:

```cpp
template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

using Command = std::variant<Move, Attack, Heal, Quit>;

void handle(const Command& cmd) {
    std::visit(overloaded{
        [](const Move& m)   { player.move_to(m.target); },
        [](const Attack& a) { combat.resolve(a); },
        [](const Heal& h)   { player.hp += h.amount; },
        [](const Quit&)     { engine.shutdown(); }
    }, cmd);
}
```

This is a closed-set, value-semantic alternative to virtual dispatch — exhaustiveness is checked at compile time.

---

## 10. Contracts (C++26)

C++26 contracts bring Design-by-Contract as a language feature, replacing `assert()` macros and ad-hoc precondition checks.

```cpp
class CircularBuffer {
    int pop()
        pre(size_ > 0)
        post(r: r >= min_value_ && r <= max_value_)
    {
        --size_;
        return buffer_[head_++ % capacity_];
    }

    void push(int value)
        pre(size_ < capacity_)
        pre(value >= min_value_ && value <= max_value_)
        post(size_ > 0)
    {
        buffer_[tail_++ % capacity_] = value;
        ++size_;
    }
};
```

Contracts can be checked, assumed, or ignored per build configuration — enabling both defensive programming in debug and zero-overhead in release.

---

## Summary Table

|Pattern|Replaces|Key Feature|
|---|---|---|
|Deducing `this`|CRTP|`this auto&&`|
|Monadic errors|Exceptions / error codes|`std::expected`|
|Lazy sequences|Iterator boilerplate|`std::generator`|
|Multidim views|Raw pointer math|`std::mdspan`|
|Compile-time strategy|`if constexpr` hacks|`if consteval`|
|Type-erased dispatch|Inheritance hierarchies|`std::move_only_function`|
|Auto-serialization|Macros / codegen|Static reflection (`^T`)|
|Structured concurrency|Raw threads / callbacks|Senders/receivers|
|Variant dispatch|Virtual functions|`std::visit` + `overloaded`|
|Design by contract|`assert()` macros|`pre` / `post`|

---

_These patterns represent a shift in modern C++ philosophy: prefer value semantics, compose at compile time, and let the type system enforce correctness — with zero runtime cost._