## Opinionated Principles for Writing Excellent C++23/26

*For any performance-aware, production-grade system. Not domain-specific.*

---

## 1. Repository & Build Structure

### Directory Layout

```
project/
├── CMakeLists.txt
├── cmake/
│   ├── CompilerWarnings.cmake
│   ├── Sanitizers.cmake
│   └── StaticAnalysis.cmake
├── src/
│   ├── core/
│   │   ├── types.h                  # Fundamental value types, aliases
│   │   ├── assert.h                 # Custom assertion macros
│   │   ├── memory_arena.h           # Arena allocator (header-only, template)
│   │   ├── memory_arena_test.cpp
│   │   ├── ring_buffer.h            # SPSC lock-free ring buffer
│   │   ├── ring_buffer_test.cpp
│   │   └── time.h                   # Clock wrappers
│   ├── transport/
│   │   ├── tcp_session.h
│   │   ├── tcp_session.cpp
│   │   ├── tcp_session_test.cpp
│   │   ├── udp_socket.h
│   │   ├── udp_socket.cpp
│   │   └── udp_socket_test.cpp
│   ├── storage/
│   │   ├── write_ahead_log.h
│   │   ├── write_ahead_log.cpp
│   │   ├── write_ahead_log_test.cpp
│   │   ├── page_cache.h
│   │   ├── page_cache.cpp
│   │   └── page_cache_test.cpp
│   ├── protocol/
│   │   ├── message_codec.h
│   │   ├── message_codec.cpp
│   │   ├── message_codec_test.cpp
│   │   ├── framing.h
│   │   └── framing_test.cpp
│   ├── app/
│   │   ├── server.h
│   │   ├── server.cpp
│   │   ├── server_test.cpp
│   │   ├── config.h
│   │   └── config.cpp
│   └── main.cpp
├── bench/
│   ├── ring_buffer_bench.cpp
│   ├── codec_bench.cpp
│   └── page_cache_bench.cpp
├── tools/
│   ├── traffic_generator.cpp
│   └── log_inspector.cpp
├── config/
│   └── defaults.toml
├── scripts/
│   └── pin_cores.sh
└── docs/
    ├── architecture.md
    └── data_flow.md
```

### Key Structural Decisions

**Tests live next to source.** `tcp_session_test.cpp` sits beside `tcp_session.cpp`. No separate `tests/` mirror tree. Why: proximity reduces friction, makes it obvious when tests are missing, and you can grep a single directory to see the full picture. A separate `tests/` tree drifts from source — files get added without tests and nobody notices for weeks.

**No separate `include/` directory.** Headers live with their `.cpp` files. Why: a separate `include/` tree exists to serve *external consumers* of a library. If you're building an application, everything is internal. If you later extract a library, create the `include/` boundary then. Don't pay for a distinction you don't need yet.

**Flat within modules, deep across modules.** Each subsystem (`transport/`, `storage/`) is a flat list of files. No nested subdirectories within a module. Why: nesting creates navigation overhead, hides files, and encourages premature abstraction. If a module grows past ~15 files, split the module, not add subdirectories.

### Modules vs Headers

**Use traditional headers, not C++20 modules.** As of C++23/26, module toolchain support (across compilers, build systems, sanitizers, IDE tooling) remains inconsistent enough that adopting them in a production system is a reliability risk. The compile-time gains are real but not worth debugging mysterious module partition failures at 2am before a deploy.

Revisit when: your entire toolchain (compiler, CMake/Bazel, clang-tidy, sanitizers, debugger) handles modules without friction for 6+ months.

### Header-Only When

Header-only for:
- Templates (no choice).
- Small, pure utility functions that inline (< 10 lines).
- Data-only types (`struct Config { ... };`).

Separate `.cpp` when:
- Implementation exceeds ~30 lines.
- Has non-trivial dependencies (system calls, I/O, heavy STL).
- You want compilation firewalls — changing `tcp_session.cpp` shouldn't recompile `storage/`.

```cpp
// types.h — header-only, pure data, no dependencies
#pragma once
#include <cstdint>

enum class Status : uint8_t { kOk = 0, kError = 1 };

struct ByteSpan {
    const uint8_t* data;
    uint32_t       length;
};
```

### Static vs Shared Linking

**Static linking by default.** Why:
- One binary. Deploy by copying a file. No `LD_LIBRARY_PATH` surprises.
- The linker sees everything, enabling whole-program optimization (LTO).
- No PLT/GOT indirection on function calls — measurable in tight loops.
- Startup is faster (no dynamic loader resolution).
- Reproducible builds: the binary *is* the release artifact.

Shared linking is for plugin systems and OS libraries. Build shared only when you have a concrete plugin use-case.

### Build System

**CMake.** Not because it's wonderful — it's the least-bad option with the widest tooling support. Bazel is superior for large monorepos but adds operational complexity that's unjustified for a single-application repository.

```cmake
# cmake/CompilerWarnings.cmake
function(set_strict_warnings target)
    target_compile_options(${target} PRIVATE
        -Wall -Wextra -Wpedantic -Werror
        -Wshadow -Wconversion -Wsign-conversion
        -Wnon-virtual-dtor -Wold-style-cast
        -Wcast-align -Wunused -Woverloaded-virtual
        -Wnull-dereference -Wdouble-promotion
        -Wformat=2 -Wimplicit-fallthrough
    )
endfunction()
```

### Dependency Policy

**Minimize external dependencies, especially on the critical path.** Allowed:
- C++ standard library (selectively — see what to avoid in §9).
- OS/platform APIs where necessary.
- A config parser (cold path only).
- Google Test + Google Benchmark (test/bench only, never linked into prod).

Forbidden unless justified in writing:
- Boost (too large, too many implicit allocations, impossible to audit as a whole).
- Any "utility" library that reimplements 50 lines you could write yourself.

**Prevent dependency creep:** Every new dependency requires a written justification answering: (1) What does this do that we can't write in < 500 lines? (2) What is its allocation/exception/thread-safety model? (3) Who maintains it and what's their release cadence? If anyone on the team can write it in a day, write it.

---

## 2. Translation Units & Physical Design

### Compilation Boundaries

Each module (`transport/`, `storage/`, etc.) is a compilation boundary. Files within a module may freely include each other. Files across modules include only the other module's public headers.

The dependency graph must be a DAG:

```
core/ ← transport/ ← protocol/
  ↑         ↑            ↑
  └─────────┴── storage/ ┴── app/
```

Enforce this in CMake with `target_link_libraries` — if `protocol` doesn't link `storage`, it can't include storage headers. The build system *is* the architecture enforcer.

### Minimize Compile Times

1. **Forward-declare aggressively.** If a header only uses `Server*`, forward-declare. Don't include `server.h`.
2. **Keep headers minimal.** A header declares the interface. The `.cpp` includes the world. Common mistake: including `<algorithm>` in a header because you used `std::min` in an inline function. Move it out of the header.
3. **Precompiled headers for test frameworks only.** GTest headers are enormous. PCH them. Don't PCH your own code — it masks dependency problems.
4. **Unity builds as a CI option, not the default.** They speed up full rebuilds but destroy incremental builds and hide ODR violations.

### What Goes Where

**Header (`.h`):**
- Type definitions (structs, enums, concepts).
- Function declarations.
- Inline/template definitions.
- Constants (`constexpr`).
- Static assertions.

**Implementation (`.cpp`):**
- Function bodies.
- File-local helpers (anonymous namespace).
- System calls, heavy STL usage.
- The "why" comments for non-obvious implementations.

```cpp
// page_cache.h — the interface contract
#pragma once
#include "core/types.h"
#include <cstdint>

class PageCache {
public:
    struct Config {
        uint32_t page_size_bytes;
        uint32_t page_count_max;
    };

    explicit PageCache(Config config);

    // Fetch a page by index. Returns a view into the cache.
    // Precondition: page_index < config.page_count_max.
    [[nodiscard]] ByteSpan get_page(uint32_t page_index) const;

    // Mark a page as dirty (needs writeback).
    // Precondition: page_index < config.page_count_max.
    void mark_dirty(uint32_t page_index);

    [[nodiscard]] uint32_t dirty_count() const;

private:
    Config config_;
    // ... storage details hidden in .cpp
};
```

---

## 3. Core Design Philosophy

### Value Semantics by Default

Everything is a value unless it must be shared or polymorphic. Values are easy to reason about: they don't alias, they copy cheaply (or you make them non-copyable), and they're trivially safe across threads.

```cpp
// Good: value type, trivially copyable, no surprises.
struct LogEntry {
    uint64_t timestamp_ns;
    uint32_t sequence_number;
    uint16_t payload_length;
    uint8_t  severity;
    uint8_t  source_id;
};
static_assert(std::is_trivially_copyable_v<LogEntry>);
```

Pass by `const&` for reads, by value if you need a copy, by pointer-to-non-const for out parameters on large structs.

### Ownership Model

**Ownership is decided at architecture time, not coding time.**

- **Arenas for bulk homogeneous data.** Message buffers, page caches, object pools — allocate from a pre-sized arena at startup. Arena outlives all references. No ref-counting, no shared ownership debate.
- **`unique_ptr` for subsystem components.** `main()` owns the major components. They're created once, destroyed at shutdown.
- **Raw pointers/references for non-owning access.** A `Processor` gets a `const Cache&` — it doesn't own it, doesn't extend its lifetime, just reads.
- **`shared_ptr` is a last resort.** If you're reaching for `shared_ptr`, stop and reconsider the ownership model first. It indicates ambiguity in who owns the resource. There are legitimate uses (truly shared, multi-owner lifetimes), but they're rarer than people think.

```cpp
int main() {
    // All major allocations happen here, at startup, explicitly.
    auto config = load_config("config/defaults.toml");

    auto arena = MemoryArena(config.arena_size_bytes);
    auto transport = std::make_unique<TcpSession>(config.transport);
    auto cache = std::make_unique<PageCache>(config.cache);
    auto codec = std::make_unique<MessageCodec>(config.protocol);

    // Wire: non-owning references. Lifetimes are clear.
    auto server = Server(*transport, *cache, *codec);

    server.run();
}
```

### Error Handling: Pick a Strategy, Be Consistent

There are three defensible strategies. Choose one for your project and apply it uniformly:

**Strategy A: Exceptions disabled (`-fno-exceptions`) + `std::expected` + assertions.** Best for latency-sensitive systems where you want zero hidden control flow and zero landing-pad overhead.

**Strategy B: Exceptions for truly exceptional failures + error codes for expected conditions.** Viable for systems where the "exceptional" case (out of memory, disk full) truly is rare and the hot path never throws.

**Strategy C: Exceptions everywhere (standard C++ style).** Acceptable for applications where latency tails don't matter and developer velocity is paramount.

This guide favors **Strategy A** because it makes control flow fully explicit:

```cpp
enum class ConnectError : uint8_t {
    kDnsResolutionFailed,
    kConnectionRefused,
    kTimeout,
};

// Fallible operation: connection setup. Failure is expected.
[[nodiscard]] std::expected<TcpSession, ConnectError>
connect_to_server(const Endpoint& endpoint, Duration timeout);

// Hot path: page access. Invalid index is a bug, not an error.
ByteSpan PageCache::get_page(uint32_t page_index) const {
    assert(page_index < config_.page_count_max
           && "page_index out of bounds");
    // ... fast path, no error return needed
}
```

Whichever strategy you pick, the rule is: **expected failures are handled in normal control flow; bugs are caught by assertions.**

### Immutability Defaults

Mark everything `const` unless mutation is required. Parameters, locals, member functions, data members if feasible.

```cpp
[[nodiscard]] uint64_t compute_checksum(
    const uint8_t* const data,
    const uint32_t length)
{
    assert(data != nullptr);
    assert(length > 0);

    uint64_t hash = kInitialSeed;
    for (uint32_t i = 0; i < length; ++i) {
        hash = hash_combine(hash, data[i]);
    }
    return hash;
}
```

### Dynamic Allocation Policy

The spectrum ranges from "never after init" (hard real-time) to "freely, but mindfully" (most applications). The principle is universal: **know when you allocate and why.**

At minimum:
- Pre-size containers when you know the count. `reserve()` before a loop.
- Avoid allocating in tight loops.
- Prefer stack allocation for small, short-lived objects.
- Use arenas for bulk allocations of similar-lifetime objects.

For hard real-time or latency-sensitive paths, go further: allocate everything at startup and run allocation-free afterward. You can enforce this in debug builds:

```cpp
// Debug builds only: detect accidental allocations on hot path.
inline std::atomic<bool> g_allocations_frozen{false};

void* operator new(std::size_t size) {
    assert(!g_allocations_frozen.load(std::memory_order_relaxed)
           && "unexpected allocation after init");
    return std::malloc(size);
}
```

### Enforce Invariants Aggressively

Assertions are documentation that runs. Average 2+ per function. Assert preconditions, postconditions, and invariants. Assert both the positive and negative space:

```cpp
void RingBuffer::push(const uint8_t* data, uint32_t length) {
    // Preconditions
    assert(data != nullptr && "null data pointer");
    assert(length > 0 && "zero-length push");
    assert(length <= capacity_ && "data exceeds buffer capacity");

    // Invariant: write position is always valid
    assert(write_pos_ < capacity_);

    // ... implementation ...

    // Postcondition
    assert(size() <= capacity_ && "buffer overflow");
}
```

---

## 4. Class & Type Design

### What Makes a Good Type

1. **Invariants enforced at construction.** If a `Quantity` can't be zero, the constructor asserts or the factory returns `std::expected`.
2. **Small and focused.** One responsibility. If it's > 200 lines, it's probably two types.
3. **Trivially copyable when it's a value type.** `memcpy` should work. No vtables, no pointers to self, no `std::string` members in hot-path types.
4. **`const`-correct.** Readers can't mutate.
5. **No default constructor if "empty" is meaningless.** Don't create invalid states just for convenience.

### `struct` vs `class`

**`struct`** for data products — all members public, no invariants beyond what member types enforce. Use aggregate initialization.

**`class`** for types with invariants — private data, public interface. The constructor establishes the invariant; methods maintain it.

```cpp
// struct: no invariants, just data grouping.
struct Measurement {
    uint64_t timestamp_ns;
    double   value;
    uint32_t sensor_id;
    uint8_t  quality;       // 0-100
};
static_assert(std::is_trivially_copyable_v<Measurement>);
static_assert(sizeof(Measurement) <= 64);  // fits one cache line

// class: invariant (buffer is always non-null and within capacity).
class DynamicBuffer {
public:
    explicit DynamicBuffer(uint32_t capacity);
    // ... methods that maintain the invariant
private:
    std::unique_ptr<uint8_t[]> data_;
    uint32_t capacity_;
    uint32_t size_;
};
```

### When to Use What

**Concepts + templates:** When you need compile-time polymorphism — different types that share a behavioral contract. Zero overhead at runtime.

```cpp
template <typename T>
concept Serializable = requires(const T& value, uint8_t* buf,
                                uint32_t capacity) {
    { value.serialize(buf, capacity) } -> std::same_as<uint32_t>;
    { T::kSerializedSizeMax } -> std::convertible_to<uint32_t>;
};

template <Serializable T>
uint32_t write_to_buffer(const T& value, uint8_t* buffer,
                         uint32_t capacity)
{
    assert(buffer != nullptr);
    assert(capacity >= T::kSerializedSizeMax);
    return value.serialize(buffer, capacity);
}
```

**CRTP:** Static interface enforcement with shared implementation logic. The base provides structure; the derived provides behavior. No vtable.

```cpp
template <typename Derived>
class ValidatorBase {
public:
    [[nodiscard]] bool validate(const Request& request) const {
        assert(request.payload.length > 0);
        // Shared pre-checks
        if (request.payload.length > kPayloadMax) {
            return false;
        }
        // Derived-specific checks
        return static_cast<const Derived*>(this)
            ->validate_impl(request);
    }
};

class StrictValidator : public ValidatorBase<StrictValidator> {
    friend class ValidatorBase<StrictValidator>;
    bool validate_impl(const Request& req) const { /* ... */ }
};
```

**Policy-based design:** When a type has multiple orthogonal axes of variation (e.g., threading policy, allocation policy, logging policy). Each axis is a template parameter.

**Type erasure / `std::variant`:** For closed sets of types that share a protocol. Prefer `std::variant` (closed, stack-allocated, compiler-checked exhaustiveness) over `std::any` (open, heap-allocated, no compile-time safety).

**PImpl:** When you need a compilation firewall for a type whose implementation changes frequently and whose header is included widely. Costs one heap allocation and one pointer indirection. Worth it for build-time savings in large codebases.

**Virtual dispatch:** For open sets of types determined at runtime — plugins, drivers, UI widgets. Also for cold-path decisions like "which backend did the user select at startup?"

**When to eliminate `virtual`:** Whenever the set of types is closed and known at compile time, or whenever the dispatch is on a hot path. Template it.

### Class Layout Convention

```cpp
class Server {
public:
    // --- Configuration (nested types first) ---
    struct Config {
        uint32_t port;
        uint32_t worker_count;
        uint32_t connection_limit;
        Duration idle_timeout;
    };

    // --- Construction ---
    explicit Server(Config config, Transport& transport,
                    Cache& cache);

    // --- Core operations (most important first) ---
    void run();
    void shutdown();

    // --- Queries ---
    [[nodiscard]] uint32_t active_connections() const;
    [[nodiscard]] bool is_running() const;

    // --- Callbacks (last in public interface) ---
    using OnRequestCallback =
        void(*)(void* context, const Request& request);
    void set_on_request(OnRequestCallback callback, void* context);

private:
    Config     config_;
    Transport& transport_;
    Cache&     cache_;

    std::vector<Connection> connections_;  // sized at construction
    uint32_t                active_count_;

    OnRequestCallback on_request_callback_;
    void*             on_request_context_;

    // --- Private helpers (prefixed with caller name) ---
    void run_accept_new_connections();
    void run_process_ready_connections();
    void shutdown_drain_connections();
};
```

**Layout rules:**
- Public before private (interface first, implementation second).
- Within public: nested types → constructors → core operations → queries → callbacks.
- Within private: data members → helper methods.
- Group related members. Separate RAII resources with blank lines.
- Most important/frequently-called methods near the top.

---

## 5. Data Layout & Performance

### AoS vs SoA

**SoA (Struct of Arrays) when you batch-process one field at a time.** If you're scanning prices to find the minimum, you want all prices contiguous:

```cpp
// AoS: scanning values touches 24 bytes per element (wasteful)
struct Record {
    double   value;      // 8
    uint64_t timestamp;  // 8
    uint32_t source_id;  // 4
    uint32_t flags;      // 4
};
std::array<Record, 1024> records;

// SoA: scanning values touches only 8 bytes per element
struct RecordTable {
    static constexpr uint32_t kCapacity = 1024;
    std::array<double, kCapacity>   values;
    std::array<uint64_t, kCapacity> timestamps;
    std::array<uint32_t, kCapacity> source_ids;
    std::array<uint32_t, kCapacity> flags;
    uint32_t count;
};
// Scanning values: 1024 * 8 = 8 KB. Fits in L1 cache.
```

**AoS when you always access entire records together** — e.g., serializing a record to disk, sending it over the network. All fields of one record are adjacent.

**The heuristic:** If your hot loop touches 1–2 fields, SoA. If it touches all fields, AoS.

### Cache Line Alignment

The cache line is 64 bytes on x86. Structures that are hot and per-thread or contended should be aligned:

```cpp
struct alignas(64) ThreadLocalStats {
    uint64_t requests_processed;  // 8
    uint64_t bytes_received;      // 8
    uint64_t errors_count;        // 8
    uint64_t last_request_ns;     // 8
    // 32 bytes used, 32 bytes padding — one full cache line.
};
static_assert(sizeof(ThreadLocalStats) == 64);
```

### Prevent False Sharing

When two threads write to different data that shares a cache line, you get cache-line bouncing. Cost: ~100ns per access instead of ~1ns.

```cpp
// BAD: adjacent atomics on the same cache line
struct Counters {
    std::atomic<uint64_t> thread_a_count;  // thread A writes
    std::atomic<uint64_t> thread_b_count;  // thread B writes
    // Same cache line → false sharing.
};

// GOOD: pad each to its own cache line
struct alignas(64) PaddedCounter {
    std::atomic<uint64_t> value;
};
struct Counters {
    PaddedCounter thread_a_count;  // cache line 0
    PaddedCounter thread_b_count;  // cache line 1
};
static_assert(sizeof(Counters) == 128);
```

### Memory Arenas

Pre-allocate large blocks. Subsystems carve out what they need. No fragmentation, deterministic latency, simple lifetime management.

```cpp
class MemoryArena {
public:
    explicit MemoryArena(uint64_t capacity_bytes)
        : base_(static_cast<uint8_t*>(
              std::aligned_alloc(64, capacity_bytes)))
        , capacity_bytes_(capacity_bytes)
        , offset_bytes_(0)
    {
        assert(base_ != nullptr && "arena allocation failed");
        assert(capacity_bytes > 0);
        assert(capacity_bytes % 64 == 0
               && "capacity must be cache-line aligned");
    }

    ~MemoryArena() { std::free(base_); }

    MemoryArena(const MemoryArena&) = delete;
    MemoryArena& operator=(const MemoryArena&) = delete;

    template <typename T>
    [[nodiscard]] T* allocate(uint32_t count) {
        static_assert(std::is_trivially_destructible_v<T>,
                      "arena types must be trivially destructible");

        const uint64_t alignment = alignof(T);
        const uint64_t aligned =
            (offset_bytes_ + alignment - 1) & ~(alignment - 1);
        const uint64_t total = aligned + sizeof(T) * count;

        assert(total <= capacity_bytes_ && "arena exhausted");

        auto* result = reinterpret_cast<T*>(base_ + aligned);
        offset_bytes_ = total;
        return result;
    }

    void reset() { offset_bytes_ = 0; }

    [[nodiscard]] uint64_t used_bytes() const {
        return offset_bytes_;
    }

private:
    uint8_t* base_;
    uint64_t capacity_bytes_;
    uint64_t offset_bytes_;
};
```

### Lock-Free vs Locking

**Lock-free SPSC (single-producer, single-consumer) for the one genuinely critical data path** where a producer and consumer are on different threads with tight latency requirements. This is a solved problem — implement once, test exhaustively, never touch again.

**Mutex everywhere else.** An uncontended `std::mutex` costs ~25ns on x86. If 25ns matters, the question isn't "mutex vs lock-free?" — it's "should these two threads share data at all?" Eliminate sharing by partitioning data per thread.

**Avoid lock-free MPMC.** Nearly impossible to get right, hard to debug, and the performance benefit over a mutex is marginal for most use cases.

```cpp
// SPSC ring buffer — the only lock-free structure most projects need.
template <typename T, uint32_t Capacity>
    requires (std::has_single_bit(Capacity))  // power of 2
          && (std::is_trivially_copyable_v<T>)
class SpscRingBuffer {
public:
    static_assert(Capacity > 0);
    static_assert(Capacity <= (1u << 20),
                  "ring buffer unreasonably large");

    [[nodiscard]] bool try_push(const T& item) {
        const uint64_t head =
            head_.load(std::memory_order_relaxed);
        const uint64_t tail =
            tail_.load(std::memory_order_acquire);

        if (head - tail >= Capacity) {
            return false;
        }
        storage_[head & kMask] = item;
        head_.store(head + 1, std::memory_order_release);
        return true;
    }

    [[nodiscard]] bool try_pop(T* out) {
        assert(out != nullptr);

        const uint64_t tail =
            tail_.load(std::memory_order_relaxed);
        const uint64_t head =
            head_.load(std::memory_order_acquire);

        if (tail >= head) {
            return false;
        }
        *out = storage_[tail & kMask];
        tail_.store(tail + 1, std::memory_order_release);
        return true;
    }

private:
    static constexpr uint32_t kMask = Capacity - 1;

    alignas(64) std::atomic<uint64_t> head_{0};
    alignas(64) std::atomic<uint64_t> tail_{0};
    alignas(64) std::array<T, Capacity> storage_{};
};
```

### Branch Prediction & Data Locality

**Organize data so the common case is linear.** The hot loop should process the expected case straight-line; error handling and edge cases branch away.

**Use `[[likely]]` / `[[unlikely]]` sparingly and correctly:**

```cpp
std::expected<Message, DecodeError>
decode_message(const uint8_t* data, uint32_t length) {
    if (length < kHeaderSize) [[unlikely]] {
        return std::unexpected(DecodeError::kTruncated);
    }

    const auto type = read_message_type(data);
    if (is_known_type(type)) [[likely]] {
        return parse_body(data, length, type);
    } else [[unlikely]] {
        return std::unexpected(DecodeError::kUnknownType);
    }
}
```

**Sort hot data by access frequency.** In a struct, put the most-read fields first. The first 64 bytes are one cache line — accessing fields beyond that costs an extra miss.

**Avoid pointer-chasing.** `std::map`, `std::unordered_map` with bucket chains, `std::list` — all pointer-chasing nightmares. Use flat arrays with direct indexing when possible, `std::vector` sorted + `std::lower_bound` for small ordered sets, and open-addressing hash maps for larger associative needs.

---

## 6. Algorithm Selection Philosophy

### STL: When It's Enough

`std::sort`, `std::lower_bound`, `std::copy`, `std::fill`, `std::find_if` — these are fine for most code. They're well-tested, readable, and compilers optimize them well.

### STL: When It's Not

**Hot-path iteration with complex lambdas** that the compiler can't vectorize. Write the explicit loop and check the assembly.

**Searching in small collections:** For < ~64 elements, a linear scan over a flat array beats `std::unordered_map` because of cache locality and zero hashing overhead.

### Ranges in Performance-Critical Code

**Use cautiously.** `std::ranges` and views compose beautifully but generate complex template instantiations that compilers optimize *sometimes*. In a latency-sensitive path, you need *always*. For most application code, ranges are fine and improve readability. For the innermost hot loop, write the explicit loop, verify the assembly.

### Beyond Big-O

Big-O is necessary but not sufficient. Two O(n) algorithms can differ by 100x in practice.

**Back-of-envelope example (scanning 1000 records for a value):**
- Linear scan over flat array: 1000 * 8 bytes = 8 KB. Fits in L1. ~1μs.
- `std::unordered_map` lookup: hash (~3ns) + bucket chase (1+ cache miss) + key compare. ~8ns per lookup, but add ~4ns per cache miss for each chain hop.
- `std::map` lookup: log2(1000) ≈ 10 pointer traversals. 10 cache misses worst case = 10 * 4ns = 40ns. Plus comparison costs.

For small n, flat arrays win on cache behavior alone.

**Vectorization matters.** A loop processing 4 doubles per cycle (AVX2) is 4x faster than scalar. The compiler vectorizes simple loops over contiguous arrays with no data dependencies or branches. Design data layouts accordingly.

---

## 7. Design Patterns

### Essential Patterns

**Options struct for configuration:**

```cpp
// BAD: positional args are mixable and unreadable
Server(uint32_t port, uint32_t workers, bool tls,
       Duration timeout);

// GOOD: named fields, self-documenting, extensible
struct ServerConfig {
    uint32_t port;
    uint32_t worker_count;
    bool     tls_enabled;
    Duration timeout_idle;
};
Server(ServerConfig config, Transport& transport);
```

**Type-safe wrappers to prevent domain confusion:**

```cpp
// Prevent mixing IDs from different domains.
struct UserId     { uint64_t value; };
struct SessionId  { uint64_t value; };
struct RequestId  { uint64_t value; };

// Now the compiler catches: find_session(user_id) — type error.
Session* find_session(SessionId session_id);
```

**Sum types via `std::variant` for closed message sets:**

```cpp
// Algebraic data type: a command is exactly one of these.
using Command = std::variant<
    StartCommand,
    StopCommand,
    ConfigReloadCommand,
    HealthCheckCommand
>;

// Pattern match with visit — exhaustive by construction.
void handle_command(const Command& cmd) {
    std::visit(overloaded{
        [this](const StartCommand& c)        { do_start(c); },
        [this](const StopCommand& c)         { do_stop(c); },
        [this](const ConfigReloadCommand& c) { do_reload(c); },
        [this](const HealthCheckCommand& c)  { do_health(c); },
    }, cmd);
}
```

**`overloaded` helper (until the language provides it):**

```cpp
template <typename... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <typename... Ts>
overloaded(Ts...) -> overloaded<Ts...>;
```

### Harmful Patterns

- **Singleton.** Global mutable state with lazy initialization and hidden lifetime. Use explicit dependency injection — `main()` owns everything, passes references down.
- **Observer with dynamic registration.** Hidden control flow, allocation on subscribe, lifetime bugs. Pass callbacks explicitly at construction or use a typed event queue.
- **Abstract factory.** If you have 2–3 concrete types, `if/else` at startup is clearer, debuggable, and has zero overhead.
- **Strategy pattern with virtual dispatch on a hot path.** Template it instead.

### Composition Over Inheritance

Inheritance creates tight coupling and makes types non-trivial. Compose by value or reference:

```cpp
// BAD: deep inheritance
class HttpServer : public TcpServer, public Logger,
                   public Configurable { ... };

// GOOD: composition with explicit dependencies
class HttpServer {
public:
    HttpServer(TcpListener& listener, Logger& logger,
               const HttpConfig& config)
        : listener_(listener), logger_(logger),
          config_(config) {}
private:
    TcpListener& listener_;
    Logger&      logger_;
    HttpConfig   config_;
};
```

### Haskell Influence: Functional Patterns in C++

**Pure functions as the default.** A function that takes inputs and returns outputs with no side effects is trivially testable, composable, and safe to call from any context.

```cpp
// Pure: depends only on inputs, no side effects.
[[nodiscard]] double compute_moving_average(
    const double* const values,
    const uint32_t count,
    const uint32_t window_size)
{
    assert(values != nullptr);
    assert(count > 0);
    assert(window_size > 0);
    assert(window_size <= count);

    double sum = 0.0;
    const uint32_t start = count - window_size;
    for (uint32_t i = start; i < count; ++i) {
        sum += values[i];
    }
    return sum / static_cast<double>(window_size);
}
```

**`std::expected` as a monadic error channel (C++23):**

```cpp
[[nodiscard]] std::expected<Config, LoadError>
load_config(const std::string_view path) {
    auto file_content = read_file(path);
    if (!file_content) {
        return std::unexpected(LoadError{
            .phase = "read", .detail = file_content.error()});
    }

    auto parsed = parse_toml(*file_content);
    if (!parsed) {
        return std::unexpected(LoadError{
            .phase = "parse", .detail = parsed.error()});
    }

    return validate_config(*parsed);
}
```

**Centralize state, push purity down.** The top-level orchestrator manages state; leaf functions are pure. This is the functional architecture pattern — impure shell, pure core.

```cpp
// Impure shell: reads state, calls pure functions, applies effects.
void Server::on_request(const Request& request) {
    // Pure: compute the response
    const auto response = compute_response(
        request, cache_.snapshot(), config_);

    // Effect: send it
    transport_.send(response);
    stats_.requests_processed++;
}

// Pure core: no side effects, trivially testable.
[[nodiscard]] Response compute_response(
    const Request& request,
    const CacheSnapshot& cache,
    const Config& config)
{
    // ... all logic here, no I/O, no mutation of external state
}
```

### Compile-Time Disappearing Abstractions

The best abstractions generate zero overhead:

```cpp
// Strong typedef: zero cost at runtime (compiles to raw uint64_t),
// full type safety at compile time.
template <typename Tag>
struct StrongId {
    uint64_t value;

    constexpr bool operator==(const StrongId&) const = default;
    constexpr auto operator<=>(const StrongId&) const = default;
};

using UserId    = StrongId<struct UserTag>;
using SessionId = StrongId<struct SessionTag>;

// sizeof(UserId) == sizeof(uint64_t)
// No vtable, no indirection, no overhead.
static_assert(sizeof(UserId) == sizeof(uint64_t));
```

---

## 8. Testing & Verification

### Unit Tests

Every file gets a `_test.cpp`. Tests are in the same directory. Each test asserts one behavior:

```cpp
TEST(PageCache, GetPageReturnsCorrectSpan) {
    auto cache = PageCache({.page_size_bytes = 4096,
                            .page_count_max = 16});
    cache.write_page(0, test_data, sizeof(test_data));

    const auto span = cache.get_page(0);

    EXPECT_EQ(span.length, 4096u);
    EXPECT_EQ(std::memcmp(span.data, test_data,
                          sizeof(test_data)), 0);
}
```

### Property-Based Testing

For core data structures, generate random inputs and check invariants:

```cpp
TEST(RingBuffer, NeverLosesDataUnderRandomOperations) {
    SpscRingBuffer<uint64_t, 1024> rb;
    auto rng = std::mt19937(42);  // deterministic seed

    uint64_t pushed = 0;
    uint64_t popped = 0;
    uint64_t next_push_value = 0;
    uint64_t expected_pop_value = 0;

    for (uint32_t i = 0; i < 1'000'000; ++i) {
        if (rng() % 2 == 0) {
            if (rb.try_push(next_push_value)) {
                next_push_value++;
                pushed++;
            }
        } else {
            uint64_t value = 0;
            if (rb.try_pop(&value)) {
                ASSERT_EQ(value, expected_pop_value);
                expected_pop_value++;
                popped++;
            }
        }
    }
    // Drain remaining
    uint64_t value = 0;
    while (rb.try_pop(&value)) {
        ASSERT_EQ(value, expected_pop_value++);
    }
    EXPECT_EQ(pushed, expected_pop_value);
}
```

### Determinism

**All tests are deterministic.** Seed all RNGs explicitly. Mock or inject clocks. No real network I/O in unit tests. Tests must produce identical results on every run, on every machine.

### Benchmark Methodology

Use Google Benchmark with:
1. **Pinned cores.** Set CPU affinity to an isolated core.
2. **Disabled frequency scaling.** `cpupower frequency-set -g performance`.
3. **Warm cache.** The framework handles this, but verify.
4. **Report percentiles** (p50, p99, p999, max) — not just mean.

```cpp
static void BM_PageCacheRead(benchmark::State& state) {
    auto cache = PageCache({.page_size_bytes = 4096,
                            .page_count_max = 1024});
    // Fill with data
    for (uint32_t i = 0; i < 1024; ++i) {
        cache.write_page(i, test_data, 4096);
    }

    uint32_t idx = 0;
    for (auto _ : state) {
        const auto span = cache.get_page(idx % 1024);
        benchmark::DoNotOptimize(span);
        idx++;
    }
}
BENCHMARK(BM_PageCacheRead);
```

### Detect Performance Regressions

CI runs benchmarks on dedicated, stable hardware (not shared runners — those have noisy neighbors). Store results in a time-series store. Alert if p99 increases by > 10% vs. the 7-day rolling average.

### Tooling

Run all of these in CI. None are optional:

- **AddressSanitizer (`-fsanitize=address`):** Buffer overflows, use-after-free, stack overflow.
- **UndefinedBehaviorSanitizer (`-fsanitize=undefined`):** Signed overflow, null deref, misaligned access.
- **ThreadSanitizer (`-fsanitize=thread`):** Data races. Run with your multithreaded tests.
- **`perf stat` / `perf record`:** Instruction counts, cache misses, branch mispredictions.
- **`valgrind --tool=cachegrind`:** Detailed cache simulation when hunting specific bottlenecks.
- **`clang-tidy`:** Static analysis with a conservative, curated set of checks.

---

## 9. What You Avoid

### Anti-Patterns

**`std::string` on hot paths.** Every construction or concatenation can allocate. Use `std::string_view` for reads, fixed-size `char` arrays for storage, and pre-allocated buffers for formatting.

**`std::unordered_map` for small, known-size lookups.** Hash maps have unpredictable latency (rehashing), pointer chasing (bucket chains), and poor cache behavior. Use `std::array` indexed by a dense integer when possible. Use a sorted `std::vector` + `std::lower_bound` for small ordered sets.

**`std::shared_ptr` as a default.** Atomic reference counting costs ~20ns per copy, trashes cache lines across cores, and usually indicates fuzzy ownership design. Refactor ownership first.

**`std::function` on hot paths.** Allocates (for large captures), type-erases, and has a virtual call internally. Use function pointers, templates, or `std::variant` of concrete callables.

**`std::iostream`.** Slow, stateful, not thread-safe, and locale-dependent by default. Use `std::format` (C++23) or `snprintf` into a pre-allocated buffer.

**`std::list` and `std::deque` on hot paths.** Pointer chasing and fragmented memory. Use `std::vector` or ring buffers.

### Over-Engineering Traps

**Don't build a "framework."** You're building an application. Every abstraction layer costs cognitive load and runtime. If you have one protocol, don't build a "protocol abstraction layer." Write the codec directly.

**Don't genericize prematurely.** Writing `template <typename Transport, typename Codec, typename Cache>` on day one means you're designing an interface without knowing the constraints. Write the concrete code first. Extract the template when the third concrete instance forces you to. The Rule of Three applies to abstractions, not just resource management.

**Don't micro-optimize before profiling.** Get the architecture right, get the data flow right, write correct code, then measure, then optimize. Optimizing the wrong code path is wasted effort.

### Costly Abstractions

- **Virtual dispatch in tight loops:** 1 vtable lookup = 1 indirect branch = potential branch misprediction ≈ 10–20 wasted cycles.
- **`std::any`:** Dynamic allocation + type erasure + RTTI. Never on a hot path.
- **Coroutines (as of C++23):** Heap-allocated frames by default. Promising for async I/O frameworks, but verify the codegen before using in latency-sensitive paths.
- **Deep ranges pipelines:** Can prevent vectorization and generate suboptimal codegen.

### Common Mid-Level Mistakes

1. **Using a mutex when the data could be partitioned.** One thread per partition eliminates all sharing.
2. **Logging synchronously on the hot path.** Write to a ring buffer; a background thread drains it.
3. **Passing large types by value when `const&` suffices.** Measure the copy cost.
4. **Not measuring.** "I think this is faster" is not engineering. `perf stat`, cache miss rates, instruction counts. Believe the numbers.
5. **Premature `std::move` on return values.** The compiler applies copy elision (NRVO) in most cases. Adding `std::move` to a `return` statement can *prevent* NRVO. Understand when each applies.
6. **Abstract interfaces "for testability."** You don't need `ICache` to mock a cache. Template the consumer, or inject a function pointer.
7. **Allocating in a loop.** Every `push_back` that triggers reallocation is a latency spike. `reserve()` or use fixed-capacity containers.
8. **Ignoring struct padding.** `{bool, int64_t, bool}` = 24 bytes. Reorder to `{int64_t, bool, bool}` = 16 bytes. Use `-Wpadded` to catch this.
9. **Catching exceptions as flow control.** Exceptions are for exceptional situations. If you catch an exception in every iteration of a loop, it's not exceptional — it's your logic.
10. **Defaulting to `std::unordered_map` for everything.** Think about access pattern, size, and lifetime before choosing a container.

---

## Naming Convention Summary

| Entity                  | Convention            | Example                       |
|-------------------------|-----------------------|-------------------------------|
| Types, Enums            | PascalCase            | `PageCache`, `Status`         |
| Functions, Variables    | snake_case            | `compute_checksum`, `count`   |
| Member variables        | snake_case_           | `capacity_`, `offset_bytes_`  |
| Constants               | kPascalCase           | `kHeaderSize`, `kPageMax`     |
| Macros                  | SCREAMING_SNAKE_CASE  | `ASSERT_NOT_NULL`             |
| Enum values             | kPascalCase           | `Side::kBid`, `Status::kOk`  |
| Template parameters     | PascalCase            | `Derived`, `Capacity`         |

**Units and qualifiers go last:** `timeout_ms`, `buffer_size_bytes`, `latency_ns_p99`.

**No abbreviations:** `connection` not `conn`, `message` not `msg`, `configuration` not `cfg`.

**Symmetric names for related concepts:** if you have `read_buffer`, pair it with `write_buffer`, not `output_buf`.

---

## Final Word

Good C++ reads like a well-organized book: each file tells you what it does, the types tell you what's valid, the assertions tell you what's assumed, and the absence of unnecessary abstractions tells you the author was focused on solving the problem rather than building a framework. Complexity is the enemy. Every line of code is a liability. Write the simplest correct thing, measure it, and improve from there.
