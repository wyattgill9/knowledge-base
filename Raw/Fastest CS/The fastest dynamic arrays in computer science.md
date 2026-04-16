
**A C-style dynamic array using `realloc`/`mremap` on Linux with a high-performance allocator like mimalloc is the fastest possible dynamic array — achieving O(1) growth without copying for large buffers and ~1.9 ns per append for trivially copyable types.** Among drop-in replacements, Facebook's `folly::fbvector` paired with jemalloc is the fastest practical C++ vector, exploiting in-place expansion, jemalloc-aware size classes, and `memcpy`-based relocation. Across standard libraries, C++ `std::vector`, Rust `Vec<T>`, and hand-rolled C arrays perform within ~10% of each other — the language barely matters once you control the allocator. The real performance gaps come from architectural decisions: growth factor, allocator choice, trivial-relocatability awareness, and whether the runtime imposes boxing, garbage collection, or interpreter overhead.

---

## Standard library vectors: C++, Rust, and C are effectively tied

The fastest standard library dynamic arrays belong to systems languages that store elements contiguously with zero per-element overhead. On a benchmark pushing 100 million `uint64_t` values, the results tell a clear story:

| Rank | Implementation | ns per insert |
|------|-------------------------------------|---------------|
| 1 | C `libdynamic` vector (glibc) | **1.92** |
| 2 | Rust `Vec<T>` (glibc) | **2.38** |
| 3 | C `libdynamic` vector (jemalloc) | 4.47 |
| 4 | Rust `Vec<T>` (jemalloc, default) | 4.91 |
| 5 | C++ `std::vector` (GCC 4.9) | 5.01 |
| 6 | LuaJIT vector assign | 6.82 |
| 7 | Java Trove `TLongArrayList` | 14.6 |
| 8 | Java `ArrayList<Long>` | 190 |

The **allocator choice dominated language choice** — switching from glibc to jemalloc caused a larger performance swing than switching from C to Rust. When pre-allocated and reused, `std::vector` matches raw C arrays within ~10%. The Computer Language Benchmarks Game confirms Rust typically lands within **5–10%** of C/C++ on array-heavy workloads, and sometimes wins.

Managed languages pay steep penalties. Java's `ArrayList<Integer>` requires **~28–36 bytes per integer** (vs. 4 bytes in C++) due to boxing, making it roughly **40× slower** for push_back of primitives. C#'s `List<T>` avoids this problem through reified generics — `List<int>` stores values inline with zero boxing, approaching native performance. Python lists sit at the bottom: the CPython interpreter adds ~50–100× overhead, and each integer consumes **36 bytes** (8-byte pointer + 28-byte `PyLongObject`). Go slices perform well for value types with zero boxing, but garbage collector pauses and bounds checking add ~10–30% overhead versus C.

Each standard library makes different growth-factor tradeoffs. GCC/Clang `std::vector` and Rust `Vec<T>` double capacity (2×). MSVC and Java use 1.5×. Python uses a conservative ~1.125×. Go adaptively transitions from 2× (small slices) down to ~1.25× (large slices). These choices matter for memory efficiency more than raw speed — the amortized cost difference between 1.5× and 2× growth is small, but 2× growth **can never reuse previously freed memory** with a first-fit allocator, since each new allocation exceeds the sum of all prior ones.

## folly::fbvector is the fastest practical drop-in replacement

Facebook's `folly::fbvector` represents the most carefully optimized general-purpose dynamic array available. It functions as a drop-in `std::vector` replacement with four key advantages that compound into significant real-world gains.

First, **jemalloc-aware allocation**. fbvector detects jemalloc at compile time and uses `goodMallocSize()` to round allocation requests up to jemalloc's size classes, eliminating allocation slack entirely. Standard `std::vector` cannot do this because `std::allocator` provides no interface for querying the allocator's preferred sizes.

Second, **in-place expansion via `xallocx()`**. When growing, fbvector first attempts to expand the existing allocation in-place using jemalloc's `xallocx()` function. If the adjacent memory is free, the buffer grows with zero copying — an operation that `std::vector` fundamentally cannot perform because C++'s `new`/`delete` model has no equivalent of `realloc`. Testing with 256 million floats showed that jemalloc's `xallocx` avoided copies for most growth attempts on buffers above 32KB.

Third, **trivial relocatability**. Types annotated with `FOLLY_ASSUME_FBVECTOR_COMPATIBLE` are relocated using `memcpy` instead of element-wise move construction during reallocation. This extends beyond `std::is_trivially_copyable` — types like `std::unique_ptr<T>` and `std::string` are safely relocatable via bitwise copy even though the C++ standard doesn't recognize this. The pending P1144 proposal would standardize this concept.

Fourth, an **adaptive growth strategy**: fbvector doubles for small buffers (where jemalloc size classes make doubling efficient), then switches to 1.5× for larger buffers (enabling memory reuse). Facebook reported that applying similar optimization philosophy to `fbstring` produced a **1% performance improvement across their entire C++ codebase** — a staggering gain at that scale. fbvector's own documentation claims improvements that are "always non-negative, frequently significant, sometimes dramatic, and occasionally spectacular."

## Small buffer optimization eliminates the heap for short-lived vectors

When vectors are small and numerous, **the allocation itself becomes the bottleneck**, not the data operations. Small buffer optimized (SBO) vectors solve this by embedding a fixed inline buffer within the object, falling back to the heap only when capacity is exceeded.

The landscape of SBO vectors reveals different design philosophies. LLVM's `SmallVector` defaults to keeping `sizeof` near 64 bytes and uniquely offers `SmallVectorImpl<T>` — a type-erased base class independent of the inline capacity N, critical for preventing template bloat across LLVM's massive codebase. Google's `absl::InlinedVector` prioritizes being a perfect `std::vector` API drop-in but lacks type erasure. `folly::small_vector` takes memory optimization to an extreme, storing the inline/heap flag in the **highest bit of the size field** — a single-bit overhead — designed for scenarios with "billions of these vectors." The third-party `ankerl::svector` achieves a minimum `sizeof` of just **8 bytes** with 7 bytes of inline capacity and only 1 byte of overhead, compared to 24–32 bytes for competing implementations.

Benchmarks reveal nuanced tradeoffs. For push_back of 1000 ints, `std::vector` actually wins because SBO vectors pay a branch cost on every access (checking inline vs. heap). For random access, `absl::InlinedVector` was measured at **~5× slower** than `std::vector` — a surprising result likely caused by the inline/heap check interfering with compiler optimizations. Rust's `SmallVec` benchmarks showed `Vec` with pre-allocated capacity was **up to 5.5× faster** than `SmallVec` for operations like `from_slice`, leading the benchmark author to conclude that "malloc is fast enough that SmallVec's increased complexity often hurts more than its allocation savings help." The real SBO win comes in hot loops creating and destroying many small vectors, or in containers-of-containers where `vector<small_vector<int,4>>` keeps all inline buffers contiguous.

`boost::container::static_vector` deserves special mention: it provides a **fixed-capacity vector with zero heap allocation** and eliminates even the inline/heap branch that other SBO vectors require. It offers genuinely O(1) non-amortized push_back (just an increment + construction) and is being standardized as `std::static_vector` via P0843.

## Five techniques that deliver the largest speedups

The most impactful dynamic array optimizations, ranked by measured effect:

- **`realloc`/`mremap` for in-place growth**: On Linux, `mremap()` remaps virtual pages without copying physical memory, making reallocation **O(1) regardless of array size**. Testing with 256M floats showed mremap completely avoided all copies from 0 to 256MB. This is the single most powerful optimization for large arrays of trivially copyable types, but C++'s `new`/`delete` model prevents `std::vector` from using it.

- **High-performance allocators**: mimalloc delivers **13–22% overall speedup** over system malloc for allocation-heavy workloads. jemalloc provides **30% higher throughput** at 36 threads with high contention and excels at long-term fragmentation resistance. tcmalloc achieves **50× throughput of default libmalloc** for 4MB allocations. Simply linking against a better allocator — without changing any code — is the highest-leverage optimization available.

- **SIMD-accelerated memory operations**: AVX2-optimized memcpy achieves **2–20× faster copying** than scalar code for large aligned blocks, processing 256 bits per instruction. Modern `memcpy`/`memmove` implementations already use SIMD internally, but explicit alignment via `xsimd::aligned_allocator` ensures the fastest SIMD code paths. For reallocation of a 1GB vector of ints, the difference between scalar and AVX-512 copying is dramatic.

- **Trivial relocatability**: Using `memcpy` instead of element-wise move construction during reallocation provides **2–10× speedup** for the reallocation operation itself. The key limitation is that C++ lacks a standard `is_trivially_relocatable` trait — folly's `IsRelocatable` annotation and the P1144 proposal address this gap. Every type that can be bitwise-moved (including `std::string`, `std::unique_ptr`, and most standard containers) benefits.

- **`push_back_unchecked` (eliminating the capacity branch)**: Research from The Coding Nest showed that a `push_back_unchecked()` — which skips the `size == capacity` check when the caller guarantees space — delivers **up to 5.9× speedup** at small sizes on Clang (540ns → 93ns for 1000 ints) because the compiler can fully vectorize the loop. Even at 10M elements, it provides 10–30% improvement. MSVC's `std::vector` is particularly penalized, running **30–80% slower** than a simple reimplementation even in release builds.

## Academic research has produced space-optimal but slower alternatives

The theoretical computer science community has explored the fundamental limits of dynamic arrays. Brodnik, Carlsson, Demaine, Munro, and Sedgewick proved in 1999 that any dynamic array must waste at least **Ω(√N)** space to maintain O(1) amortized operations — a tight lower bound. Their two-level indexing scheme achieves this bound: **N + O(√N)** space with O(1) operations, compared to standard doubling's N + O(N) space waste. A 2022 follow-up by Tarjan and Zwick (published at SOSA 2023) produced **even more memory-efficient implementations** and established a formal space-time tradeoff: for arrays using only N + O(N^{1/r}) space, the amortized grow cost is Ω(r).

Hashed Array Trees (Sitarski, 1996) offer a practical middle ground: an index block of size √N pointing to data blocks of size √N, wasting only O(√N) storage with O(1) access through one level of indirection. Tiered Vectors (Goodrich & Kloss, 1999) provide O(N^{1/k}) insertion anywhere in the array — useful when mid-array insertion matters.

However, **empirical evaluation consistently shows standard doubling arrays are faster in practice**. Katajainen's 2016 benchmarks found that O(1) worst-case dynamic array variants were "measurably slower" than `std::vector` for standard operations. The extra indirection level required by space-optimal structures defeats hardware prefetchers and increases cache misses. A sliced array with a few percent extra space was the best practical worst-case solution. The theoretical work remains valuable for memory-constrained embedded systems, but for throughput-critical applications, simple contiguous arrays with geometric growth win.

## Conclusion

The quest for the fastest dynamic array converges on a clear hierarchy. **At the absolute top sits a C-style array using `realloc` on Linux** (which invokes `mremap` for large buffers), paired with mimalloc or jemalloc — achieving sub-2ns appends and zero-copy growth for large allocations. Among production-quality C++ containers, **`folly::fbvector` with jemalloc** is the fastest general-purpose implementation, combining in-place expansion, allocator-aware sizing, and trivial relocatability. Standard `std::vector` and Rust's `Vec<T>` perform within 10% of each other and within 2–3× of the absolute minimum.

The most important insight is that **the allocator matters more than the container**. Switching from system malloc to mimalloc can deliver a 13–22% speedup with zero code changes. The second most important factor is whether the implementation can avoid copies during growth — `realloc`/`mremap` and jemalloc's `xallocx()` provide this. Growth factor (1.5× vs 2×), SBO, and `push_back_unchecked` are secondary optimizations that matter in specific scenarios. And for languages with boxing overhead (Java) or interpreter overhead (Python), the per-element cost structure dominates everything else — no amount of container cleverness can overcome a 36-byte-per-integer tax.