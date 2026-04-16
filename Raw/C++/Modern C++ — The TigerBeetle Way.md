
A comprehensive, opinionated guide to writing Modern C++ that maximizes **safety → performance → developer experience**, in that order.

---

## 1. Types & Data Representation

### Use Explicitly-Sized Integer Types — Always

Banish `int`, `long`, `unsigned`, and especially `size_t` from your vocabulary. Every integer communicates its width.

```cpp
#include <cstdint>

uint32_t sector_count = 128;
uint64_t byte_offset  = 0;
int64_t  balance_delta = -500;
uint8_t  flags        = 0;
```

**Why:** `int` is 16 bits on some embedded targets, 32 on most desktops. `size_t` is 32 or 64 depending on architecture. Explicitly-sized types eliminate an entire class of portability and overflow bugs.

The one exception: `bool`. It's unambiguous.

### Strong Types Over Naked Primitives

Two `uint64_t` arguments are an invitation for transposition bugs. Wrap them.

```cpp
// BAD — can silently swap arguments
void transfer(uint64_t debit_account_id, uint64_t credit_account_id, uint64_t amount);

// GOOD — the type system prevents mixups
struct DebitAccountID  { uint64_t value; };
struct CreditAccountID { uint64_t value; };
struct Amount          { uint64_t value; };

void transfer(DebitAccountID debit, CreditAccountID credit, Amount amount);
```

When strong types are too heavy, use an **options struct**:

```cpp
struct TransferOptions {
    uint64_t debit_account_id;
    uint64_t credit_account_id;
    uint64_t amount;
    uint32_t timeout_ms;
};

void transfer(const TransferOptions& options);

// Call site is self-documenting:
transfer({
    .debit_account_id  = 42,
    .credit_account_id = 99,
    .amount            = 1000,
    .timeout_ms        = 5000,
});
```

### Enums: Always `enum class`, Always Sized

```cpp
enum class MessageType : uint8_t {
    kPing    = 0,
    kPong    = 1,
    kRequest = 2,
    kReply   = 3,
};
```

**Why:** `enum class` prevents implicit conversion to integers and keeps values scoped. Explicit underlying type guarantees wire-format size.

### Compile-Time Constants

```cpp
static constexpr uint32_t kMaxConnections = 1024;
static constexpr uint64_t kSectorSize     = 4096;
static constexpr uint32_t kBatchMax       = 8191;

// Assert relationships between constants — this is design-level documentation
static_assert(kBatchMax < kMaxConnections, "batch must not exceed connection pool");
static_assert(kSectorSize % 512 == 0, "sector size must be disk-aligned");
static_assert(sizeof(Header) == 128, "header must be exactly 128 bytes for wire format");
```

**Use `static_assert` aggressively.** Every relationship between constants, every size assumption, every alignment requirement gets a compile-time assertion. These are free — they cost nothing at runtime and catch design-level bugs before a single line executes.

---

## 2. Containers & Data Structures

### The Cardinal Rule: No Dynamic Allocation After Initialization

All memory is allocated statically or at startup. After `init()` returns, the system's memory footprint is fixed. This means:

- No `std::vector` that grows in the hot path.
- No `std::map` or `std::unordered_map` that allocates on insert.
- No `new` / `delete` in steady-state operation.
- No `std::string` being built dynamically.

### Fixed-Capacity Containers

Build or use containers with a compile-time or startup-time upper bound:

```cpp
// A fixed-capacity ring buffer — the workhorse container
template <typename T, uint32_t Capacity>
class RingBuffer {
    static_assert(Capacity > 0, "capacity must be positive");
    static_assert((Capacity & (Capacity - 1)) == 0, "capacity must be power of two for fast modulo");

    std::array<T, Capacity> items_{};
    uint32_t head_ = 0;
    uint32_t count_ = 0;

public:
    bool push(const T& item) {
        assert(count_ <= Capacity);  // invariant: count never exceeds capacity
        if (count_ == Capacity) return false;  // operational error: full

        uint32_t index = (head_ + count_) & (Capacity - 1);  // fast modulo
        items_[index] = item;
        count_ += 1;
        return true;
    }

    std::optional<T> pop() {
        assert(count_ <= Capacity);
        if (count_ == 0) return std::nullopt;

        T item = items_[head_];
        head_ = (head_ + 1) & (Capacity - 1);
        count_ -= 1;
        return item;
    }

    [[nodiscard]] uint32_t count() const { return count_; }
    [[nodiscard]] bool empty()     const { return count_ == 0; }
    [[nodiscard]] bool full()      const { return count_ == Capacity; }
};
```

### `std::array` Over C Arrays — Always

```cpp
// BAD
uint8_t buffer[4096];

// GOOD — carries its size, works with algorithms, bounds-checkable
std::array<uint8_t, 4096> buffer{};
```

### When You Need a Flat Map: Sorted Array or Open-Addressing Hash

For static or bounded collections, prefer:

1. **Sorted `std::array` + binary search** for small collections (< ~256 elements) — cache-friendly, zero allocation.
2. **Open-addressing hash table with fixed capacity** for larger collections — no pointer chasing, no per-node allocation.

```cpp
// Small lookup table: sorted array + binary search
struct Entry {
    uint32_t key;
    uint64_t value;
};

template <uint32_t N>
class FlatLookup {
    std::array<Entry, N> entries_;
    uint32_t count_ = 0;

public:
    // Insert during init phase only
    void insert(uint32_t key, uint64_t value) {
        assert(count_ < N);
        entries_[count_] = {key, value};
        count_ += 1;
    }

    void finalize() {
        std::sort(entries_.begin(), entries_.begin() + count_,
                  [](const Entry& a, const Entry& b) { return a.key < b.key; });
    }

    std::optional<uint64_t> find(uint32_t key) const {
        auto it = std::lower_bound(
            entries_.begin(), entries_.begin() + count_,
            key, [](const Entry& e, uint32_t k) { return e.key < k; });

        if (it != entries_.begin() + count_ && it->key == key) {
            return it->value;
        }
        return std::nullopt;
    }
};
```

### `std::span` for Non-Owning Views

When passing arrays or slices to functions, never pass raw pointer + length separately. Use `std::span`:

```cpp
// BAD — two separate arguments that can go out of sync
void process(const uint8_t* data, size_t length);

// GOOD — single non-owning view, carries its own length
void process(std::span<const uint8_t> data);

// Works with any contiguous container:
std::array<uint8_t, 1024> buffer{};
process(buffer);                                    // whole array
process(std::span{buffer}.subspan(0, 512));         // first half
```

### `std::string_view` Over `const std::string&`

```cpp
// BAD — forces allocation if caller has a C string or substring
void log(const std::string& message);

// GOOD — zero-copy view, works with string literals, std::string, substrings
void log(std::string_view message);
```

### The Container Decision Tree

```
Need a fixed collection of T?
├── Size known at compile time? → std::array<T, N>
├── Size known at startup, fixed after? → std::vector<T> (reserve once, never resize)
├── Need FIFO? → RingBuffer<T, N>
├── Need key→value lookup?
│   ├── < 256 entries? → sorted std::array + binary search
│   └── Larger? → fixed-capacity open-addressing hash
├── Need a bitset? → std::bitset<N> or custom EWAH
└── Need a string? → fixed-capacity char array or std::string_view (no dynamic strings in hot path)
```

---

## 3. Control Flow

### Simple, Explicit, Flat

No recursion. No deep call chains in the hot path. Every loop has a bound.

### Conditionals: Nest, Don't Compound

```cpp
// BAD — compound condition, hard to verify all cases are handled
if (header.version == kCurrentVersion && header.checksum_valid && !header.is_corrupt) {
    process(header);
}

// GOOD — each condition is a separate gate; you can assert or handle the else of each
if (header.version != kCurrentVersion) {
    return Error::kVersionMismatch;
}
if (!header.checksum_valid) {
    return Error::kChecksumFailed;
}
if (header.is_corrupt) {
    return Error::kCorrupt;
}
process(header);
```

### State Invariants Positively

```cpp
// GOOD — reads naturally: "if the index is within bounds"
if (index < length) {
    buffer[index] = value;
} else {
    assert(false);  // unreachable: caller guarantees index < length
}

// BAD — negation is harder to reason about
if (index >= length) {
    // It's not true that we're in bounds...
}
```

### Every `if` Deserves an `else` (At Least Consider It)

If you write an `if`, ask: "What happens in the `else`?" If nothing, make that explicit:

```cpp
if (message.type == MessageType::kRequest) {
    handle_request(message);
} else {
    // All other message types are handled by the dispatch loop.
    // This branch is intentionally empty.
}
```

Or use an assertion to document and enforce:

```cpp
if (state == State::kReady) {
    begin_operation();
} else {
    assert(state == State::kIdle);  // Only Ready and Idle are valid here
}
```

### Split `else if` Chains Into Nested Trees

```cpp
// BAD — flat else-if hides the branching structure
if (a) {
    // ...
} else if (b) {
    // ...
} else if (c) {
    // ...
}

// GOOD — explicit nesting shows the decision tree
if (a) {
    // ...
} else {
    if (b) {
        // ...
    } else {
        assert(c);  // If not a and not b, c must hold.
        // ...
    }
}
```

### Switch: Exhaust All Cases, Never Default Silently

```cpp
switch (message_type) {
    case MessageType::kPing: {
        handle_ping();
        break;
    }
    case MessageType::kPong: {
        handle_pong();
        break;
    }
    case MessageType::kRequest: {
        handle_request();
        break;
    }
    case MessageType::kReply: {
        handle_reply();
        break;
    }
    // No default — if a new enum value is added, the compiler warns.
    // If you must have a default:
    // default: assert(false); // unreachable
}
```

Enable `-Wswitch-enum` to get a compiler error when a case is missing.

---

## 4. Loops

### Every Loop Has a Bound

```cpp
// GOOD — bounded by container size (which is itself bounded by a constant)
for (uint32_t i = 0; i < batch.count(); i += 1) {
    process(batch[i]);
}

// GOOD — range-based, implicitly bounded
for (const auto& item : batch) {
    process(item);
}

// BAD — unbounded
while (has_more_work()) {
    do_work();  // How long can this run? Nobody knows.
}

// GOOD — bounded version of the above
uint32_t iterations = 0;
while (has_more_work()) {
    assert(iterations < kMaxIterations);  // Fail-fast if we're stuck
    do_work();
    iterations += 1;
}
```

### Prefer `i += 1` Over `i++`

The style guide values explicitness. `i += 1` is an unambiguous statement. `i++` has a return value that is sometimes accidentally used, and `++i` vs `i++` is a perennial source of confusion.

```cpp
for (uint32_t i = 0; i < count; i += 1) {
    // ...
}
```

### Range-Based For When You Don't Need the Index

```cpp
// Preferred when index isn't needed
for (const auto& record : records) {
    assert(record.is_valid());
    process(record);
}

// Use indexed loop when you need the index for assertions or offset calculations
for (uint32_t i = 0; i < records.size(); i += 1) {
    assert(records[i].sequence == base_sequence + i);  // continuity check
    process(records[i]);
}
```

### Hot Loops: Extract to Standalone Functions

```cpp
// BAD — the optimizer must prove `this->config_` doesn't alias with `data`
void Compactor::compact(std::span<Entry> data) {
    for (uint32_t i = 0; i < data.size(); i += 1) {
        if (data[i].timestamp < config_.cutoff) {  // reload config_ every iteration?
            // ...
        }
    }
}

// GOOD — pure function, no `this`, primitive args, optimizer's dream
static void compact_entries(
    std::span<Entry> data,
    uint64_t cutoff_timestamp
) {
    for (uint32_t i = 0; i < data.size(); i += 1) {
        if (data[i].timestamp < cutoff_timestamp) {
            // ...
        }
    }
}

void Compactor::compact(std::span<Entry> data) {
    compact_entries(data, config_.cutoff);
}
```

### Batch Processing: Sequential Access, Predictable Patterns

```cpp
// GOOD — sequential access, prefetcher-friendly
void checksum_batch(std::span<const Record> records, std::span<uint32_t> checksums) {
    assert(records.size() == checksums.size());

    for (uint32_t i = 0; i < records.size(); i += 1) {
        checksums[i] = compute_checksum(records[i]);
    }
}

// BAD — random access through an indirection table
void checksum_scattered(
    std::span<const Record> records,
    std::span<const uint32_t> indices,
    std::span<uint32_t> checksums
) {
    for (uint32_t i = 0; i < indices.size(); i += 1) {
        checksums[i] = compute_checksum(records[indices[i]]);  // cache miss on every iteration
    }
}
```

---

## 5. Functions

### Shape: Inverse Hourglass

A few parameters in, a simple return type out, lots of logic in between.

```cpp
// GOOD shape — few params, simple return, meaty body
void write_batch(std::span<const Record> records) {
    assert(!records.empty());
    assert(records.size() <= kBatchMax);

    for (const auto& record : records) {
        assert(record.is_valid());
    }

    // ... 40-60 lines of real work ...
}
```

### Hard Limit: 70 Lines

If a function exceeds 70 lines, split it. But split it _well_:

- **Parent function:** owns control flow (`if`, `switch`, loop structure).
- **Helper functions:** pure computations, no branching on external state.
- **Prefix helpers with the parent name:** `write_batch()` → `write_batch_validate()`, `write_batch_encode()`.

```cpp
// Parent: owns control flow and state
void write_batch(std::span<const Record> records) {
    assert(!records.empty());
    assert(records.size() <= kBatchMax);

    write_batch_validate(records);

    auto encoded = write_batch_encode(records, buffer_);

    if (encoded.size() > kSectorSize) {
        write_batch_split(encoded);
    } else {
        storage_.write(encoded);
    }
}

// Helper: pure validation, no control flow decisions
static void write_batch_validate(std::span<const Record> records) {
    for (const auto& record : records) {
        assert(record.is_valid());
        assert(record.timestamp > 0);
    }
}

// Helper: pure computation, returns result
static std::span<const uint8_t> write_batch_encode(
    std::span<const Record> records,
    std::span<uint8_t> buffer
) {
    // ... encoding logic ...
}
```

### Assertion Density: Minimum Two Per Function

```cpp
uint64_t calculate_offset(uint32_t sector_index, uint32_t sector_count) {
    assert(sector_index < sector_count);           // precondition
    assert(sector_count <= kMaxSectors);            // precondition

    uint64_t offset = static_cast<uint64_t>(sector_index) * kSectorSize;

    assert(offset < static_cast<uint64_t>(sector_count) * kSectorSize);  // postcondition
    return offset;
}
```

### Return Type Hierarchy

Prefer simpler return types. Every layer of optionality or error-ness adds branching at the call site.

```
void                          ← best: no failure possible
bool                          ← one failure mode
uint64_t                      ← single value, sentinel not needed
std::optional<uint64_t>       ← value or nothing
std::expected<uint64_t, Error> ← value or typed error (C++23)
```

### Assertions vs. Operational Errors

This distinction is critical:

```cpp
// ASSERTION — programmer error, must never happen, crash on violation
void process_message(const Message& msg) {
    assert(msg.header.magic == kMagicNumber);  // If this fails, our code is wrong.
}

// OPERATIONAL ERROR — expected, must be handled gracefully
std::expected<Record, Error> read_record(uint64_t id) {
    auto data = storage_.read(id);
    if (!data) {
        return std::unexpected(Error::kNotFound);  // Disk might not have it. That's normal.
    }
    // ...
}
```

**Assertions crash.** That's the point. A corrupt program must not continue.

### No Exceptions

Exceptions break local reasoning, introduce invisible control flow, and make it impossible to guarantee bounded execution time. Use `std::expected` (C++23) or a similar result type:

```cpp
// Return errors explicitly
std::expected<Connection, Error> connect(std::string_view address, uint32_t port) {
    auto socket = create_socket();
    if (!socket) {
        return std::unexpected(Error::kSocketFailed);
    }

    auto result = socket->connect(address, port);
    if (!result) {
        return std::unexpected(Error::kConnectionRefused);
    }

    return Connection{std::move(*socket)};
}

// Caller handles explicitly — no hidden control flow
auto conn = connect("127.0.0.1", 3001);
if (!conn) {
    log_error("Connection failed", conn.error());
    return;
}
use_connection(*conn);
```

---

## 6. Memory & Ownership

### Static Allocation at Startup

```cpp
class Server {
    // All memory decided at construction — no further allocation.
    std::array<Connection, kMaxConnections> connections_{};
    RingBuffer<Message, kMessageQueueSize>  inbox_{};
    std::array<uint8_t, kBufferPoolSize>    buffer_pool_{};

    uint32_t active_connections_ = 0;

public:
    static std::expected<void, Error> init(Server* target, const Config& config) {
        // Initialize in-place. After this returns, memory layout is final.
        *target = Server{};
        // ... setup ...
        return {};
    }
};
```

### In-Place Initialization Pattern

```cpp
struct Journal {
    std::array<Entry, kJournalSize> entries;
    uint32_t head;
    uint32_t count;

    // Initialize through out-pointer — no intermediate copies
    static std::expected<void, Error> init(Journal* target, std::span<const uint8_t> snapshot) {
        target->head  = 0;
        target->count = 0;

        if (!snapshot.empty()) {
            auto result = deserialize(snapshot, target->entries);
            if (!result) return std::unexpected(result.error());
            target->count = *result;
        }

        assert(target->count <= kJournalSize);
        return {};
    }
};

// Usage
Journal journal;
auto result = Journal::init(&journal, snapshot_data);
```

### RAII for All Resource Lifetimes

Group resource acquisition with its cleanup scope. Separate from surrounding logic with newlines.

```cpp
void process_file(std::string_view path) {
    // Resource acquisition + scope — visually grouped
    auto file = open_file(path);
    assert(file.is_open());

    // Logic — separated by newline
    auto header = read_header(file);
    assert(header.is_valid());
    // ... process ...

}  // file closed here by RAII — no leak possible
```

### Pass by Value Only When Copying Is Intended

```cpp
// Pass by const reference — no copy, read-only access
void inspect(const Record& record);

// Pass by reference — no copy, mutation intended
void update(Record& record);

// Pass by value — explicit copy, caller knows a copy is made
void enqueue(Record record);  // Takes ownership of a copy

// Pass std::span for non-owning views of contiguous data
void process_batch(std::span<const Record> records);
```

### Zeroing and Buffer Bleeds

Always zero buffers that may be partially filled:

```cpp
std::array<uint8_t, kSectorSize> sector{};  // Zero-initialized

// After writing only part of the sector:
uint32_t bytes_written = serialize_into(sector);
assert(bytes_written <= kSectorSize);

// Explicitly zero the padding to prevent information leaks
if (bytes_written < kSectorSize) {
    std::memset(sector.data() + bytes_written, 0, kSectorSize - bytes_written);
}
```

---

## 7. Arithmetic & Off-By-One Safety

### Distinguish Index, Count, Size

```cpp
uint32_t sector_index = 5;     // 0-based position
uint32_t sector_count = 10;    // 1-based quantity: index + 1 to go from last index to count
uint64_t sector_size  = sector_count * kSectorSize;  // count × unit = size in bytes

// Converting between them:
assert(sector_index < sector_count);                  // index → count: index is always < count
uint32_t count_from_index = sector_index + 1;         // index → count: add one
uint64_t offset_from_index = static_cast<uint64_t>(sector_index) * kSectorSize;  // index → byte offset
```

### Saturating Arithmetic

Use saturating operations to prevent overflow/underflow:

```cpp
#include <numeric>  // C++26: std::add_sat, std::sub_sat, std::mul_sat, std::div_sat

// Until C++26, write your own:
constexpr uint64_t add_sat(uint64_t a, uint64_t b) {
    uint64_t result = a + b;
    if (result < a) return UINT64_MAX;  // overflow
    return result;
}

constexpr uint64_t sub_sat(uint64_t a, uint64_t b) {
    if (b > a) return 0;  // underflow
    return a - b;
}
```

### Show Arithmetic Intent

```cpp
// BAD — is this integer division intentional? Will it truncate?
uint32_t pages = byte_count / kPageSize;

// GOOD — explicit about rounding behavior
uint32_t pages_floor = byte_count / kPageSize;
uint32_t pages_ceil  = (byte_count + kPageSize - 1) / kPageSize;

assert(pages_ceil * kPageSize >= byte_count);  // postcondition: enough pages
```

### Widening Casts Are Explicit

```cpp
uint32_t sector_index = 42;
// BAD — implicit widening, easy to miss
uint64_t offset = sector_index * kSectorSize;  // if kSectorSize is uint32_t, this overflows!

// GOOD — explicit widening before multiplication
uint64_t offset = static_cast<uint64_t>(sector_index) * kSectorSize;
```

---

## 8. Struct Layout & Design

### Fields First, Then Types, Then Methods

```cpp
struct Replica {
    // — Fields —
    uint32_t id_;
    uint32_t cluster_;
    State    state_;

    // — Nested types —
    enum class State : uint8_t {
        kIdle    = 0,
        kReady   = 1,
        kActive  = 2,
    };

    // — Methods —
    static std::expected<void, Error> init(Replica* target, uint32_t id, uint32_t cluster);
    void tick();
    void on_message(const Message& msg);
};
```

### Align Names for Scanability

```cpp
struct Header {
    uint32_t checksum;       // CRC-32C of the payload
    uint32_t checksum_body;  // CRC-32C of the body only
    uint64_t timestamp;      // Nanoseconds since epoch
    uint32_t command;        // Operation type
    uint32_t size;           // Total message size in bytes
};
```

### Padding & Size Assertions

```cpp
struct WireMessage {
    uint64_t id;
    uint32_t type;
    uint32_t length;
    std::array<uint8_t, 48> payload;
};

static_assert(sizeof(WireMessage) == 64, "wire message must be exactly 64 bytes");
static_assert(alignof(WireMessage) == 8, "wire message must be 8-byte aligned");
static_assert(std::is_trivially_copyable_v<WireMessage>, "wire types must be trivially copyable");
```

---

## 9. Naming In Practice

### Units Last, Descending Significance

```cpp
uint64_t latency_ms_max = 500;
uint64_t latency_ms_min = 1;
uint64_t latency_ms_p99 = 50;

uint32_t timeout_seconds_connect = 30;
uint32_t timeout_seconds_request = 10;

// These align beautifully:
// latency_ms_max
// latency_ms_min
// latency_ms_p99
```

### Callback Naming: Parent Name as Prefix

```cpp
void read_sector(uint32_t sector_index, Callback callback);
void read_sector_callback(const Result& result);  // Named after its caller
void read_sector_validate(std::span<const uint8_t> data);  // Helper, same prefix
```

### Same-Length Related Names

```cpp
// GOOD — source/target both 6 chars, everything aligns
void copy_region(
    std::span<const uint8_t> source,
    std::span<uint8_t>       target,
    uint64_t                 source_offset,
    uint64_t                 target_offset,
    uint64_t                 source_length
) {
    assert(source_offset + source_length <= source.size());
    assert(target_offset + source_length <= target.size());
    std::memcpy(target.data() + target_offset, source.data() + source_offset, source_length);
}

// BAD — src/dest different lengths, misaligned, harder to scan
void copy_region(const uint8_t* src, uint8_t* dest, uint64_t src_off, uint64_t dest_off, uint64_t len);
```

---

## 10. Compiler & Build Discipline

### Maximum Warnings, Zero Tolerance

```cmake
target_compile_options(${PROJECT_NAME} PRIVATE
    -Wall
    -Wextra
    -Wpedantic
    -Werror                   # Warnings are errors
    -Wconversion              # Implicit narrowing
    -Wsign-conversion         # Signed/unsigned mix
    -Wshadow                  # Variable shadowing
    -Wswitch-enum             # Missing enum cases
    -Wnull-dereference
    -Wdouble-promotion
    -Wformat=2
    -Wno-unused-parameter     # Only if needed during development
)
```

### clang-format Configuration

```yaml
# .clang-format
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
AllowShortIfStatementsOnASingleLine: WithoutElse
AllowShortLoopsOnASingleLine: false
AlwaysBreakTemplateDeclarations: Yes
BraceWrapping:
  AfterFunction: false
  AfterControlStatement: Never
PointerAlignment: Left
```

---

## 11. Patterns Summary

|Pattern|When|Example|
|---|---|---|
|Options struct|≥ 2 args of same type|`TransferOptions`|
|In-place init|Large structs|`static init(T* target)`|
|Paired assertions|Data crossing boundaries|Assert before write AND after read|
|Extracted hot loop|Performance-critical inner loop|Standalone function, primitive args|
|Callback-last|Any function taking a callback|`read(data, options, callback)`|
|Parent-prefixed helpers|Splitting a large function|`compact()`, `compact_validate()`, `compact_encode()`|
|Bounded containers|All collections|Fixed-size `std::array`, `RingBuffer<T, N>`|
|Exhaustive switch|All enums|No `default`, compiler warns on missing case|
|Positive invariants|All conditionals|`if (index < length)` not `if (!(index >= length))`|
|Explicit options to library calls|All library usage|`std::stoi(input, nullptr, 10)`|

---

## 12. A Complete Example

Pulling it all together:

```cpp
#include <array>
#include <cassert>
#include <cstdint>
#include <cstring>
#include <expected>
#include <span>

static constexpr uint32_t kBatchMax    = 256;
static constexpr uint32_t kSectorSize  = 4096;
static constexpr uint32_t kSectorMax   = 1024;

static_assert(kBatchMax <= kSectorMax, "batch must fit within sector pool");
static_assert(kSectorSize % 512 == 0, "sector size must be disk-aligned");

enum class Error : uint8_t {
    kInvalidRecord = 1,
    kBatchFull     = 2,
    kStorageWrite  = 3,
};

struct Record {
    uint64_t id;
    uint64_t timestamp;
    uint64_t amount;
    uint32_t flags;
    uint32_t padding_;

    [[nodiscard]] bool is_valid() const {
        return id > 0 && timestamp > 0;
    }
};

static_assert(sizeof(Record) == 32, "record must be 32 bytes for alignment");
static_assert(std::is_trivially_copyable_v<Record>);

// ── Batch Writer ─────────────────────────────────────────────────

class BatchWriter {
    std::array<Record, kBatchMax> records_{};
    uint32_t count_ = 0;

public:
    static std::expected<void, Error> init(BatchWriter* target) {
        *target = BatchWriter{};
        assert(target->count_ == 0);
        return {};
    }

    std::expected<void, Error> add(const Record& record) {
        assert(count_ <= kBatchMax);  // invariant

        if (!record.is_valid()) {
            return std::unexpected(Error::kInvalidRecord);  // operational error
        }
        if (count_ == kBatchMax) {
            return std::unexpected(Error::kBatchFull);  // operational error
        }

        records_[count_] = record;
        count_ += 1;

        assert(count_ <= kBatchMax);  // postcondition
        return {};
    }

    [[nodiscard]] std::span<const Record> records() const {
        return std::span{records_}.subspan(0, count_);
    }

    [[nodiscard]] uint32_t count() const { return count_; }
    [[nodiscard]] bool     empty() const { return count_ == 0; }
    [[nodiscard]] bool     full()  const { return count_ == kBatchMax; }
};
```

This is the style. Safety first, performance by design, clarity by discipline.