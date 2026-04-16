**DDSketch achieves the highest raw insertion throughput at ~100M+ items/second with O(1) constant-time operations**, making it the fastest streaming approach for constructing equi-depth histograms. For exact computation, SIMD-vectorized sorting (Google Highway VQSort at **1.1 GB/s on AVX-512**) followed by boundary selection is the fastest CPU path, while GPU radix sort delivers another 10–50× on top of that. The choice between exact and approximate methods—and across the dozen major implementations—depends on whether you need guaranteed correctness, streaming capability, or distributed mergeability.

Equi-depth histograms partition data so each bin holds roughly the same count of elements. They're foundational to database query optimizers, monitoring systems, and ML feature engineering. Constructing them efficiently reduces to solving the multi-quantile problem: find the values at ranks n/B, 2n/B, …, (B−1)n/B for B bins. This report covers every major implementation across languages, databases, and algorithmic families, ranked by speed.

## Exact algorithms: from O(n log n) sorting to O(n log B) multi-selection

The baseline exact approach—sort, then pick evenly spaced elements—costs O(n log n) but benefits enormously from modern SIMD implementations. **Google Highway's VQSort** achieves **1,123 MB/s on Skylake AVX-512** for 32-bit keys (9–19× faster than `std::sort`), meaning 10M doubles (~80 MB) sort in roughly 71 ms. Intel's **x86-simd-sort** (now the NumPy sorting backend) delivers comparable throughput using AVX-512 compress-store instructions and bitonic sorting networks for small arrays. On GPUs, **NVIDIA CUB's radix sort** processes hundreds of millions of keys per second, sorting 10M doubles in 5–10 ms on an A100 including transfer overhead.

The theoretically optimal exact approach avoids full sorting entirely. The **C++ P2375 `multi_nth_element` proposal** introduces simultaneous multi-selection via recursive partitioning, achieving **O(n log B)** total work for B bin boundaries—each partition step splits both data and target ranks. For small B (say 10–20 bins), repeated calls to **Floyd-Rivest selection** (implemented in the [miniselect](https://github.com/danlark1/miniselect) library, already integrated into ClickHouse) are extremely fast at O(n) per quantile with tiny constants. Rust's `slice::select_nth_unstable()` provides similar introselect performance in its standard library.

|Method|Complexity|~Time (10M doubles)|Best for|
|---|---|---|---|
|GPU radix sort + pick|O(n), massive parallelism|**5–10 ms**|>10M elements with GPU|
|SIMD sort (VQSort/AVX-512) + pick|O(n log n), vectorized|**~71 ms**|Large arrays, CPU|
|`multi_nth_element` (P2375)|O(n log B)|~30–50 ms|Any size, optimal CPU|
|Floyd-Rivest × B (miniselect)|O(n·B) average|~15–25 ms per quantile|Few bins (<20)|
|`std::sort` / pdqsort + pick|O(n log n)|~200–500 ms|Portable baseline|

## Streaming sketches: DDSketch leads throughput, KLL leads guarantees

For data that arrives as a stream or exceeds memory, approximate quantile sketches dominate. You query B−1 evenly spaced ranks from the sketch to get equi-depth boundaries. The landscape breaks into five major algorithms, each with distinct tradeoffs.

**DDSketch** (Masson, Rim & Lee, VLDB 2019) from Datadog offers **O(1) insertion, O(1) quantile query, and full mergeability** with relative-error guarantees on quantile _values_—e.g., 1% relative accuracy means the true P99 of 100 ms is reported between 99–101 ms. Its dense-array store makes it the throughput champion. The Rust implementation ([sketches-ddsketch](https://github.com/mheffner/rust-sketches-ddsketch)) benchmarks at **70M+ inserts/sec and 350K merges/sec**. The 2026 QuantileFlow study confirmed DDSketch provides "the strongest throughput while preserving tail fidelity," with an optimized variant achieving 44% better throughput than Datadog's reference implementation. Available in Python, Java, Go, and Rust.

**KLL Sketch** (Karnin-Lang-Liberty, FOCS 2016) is the gold standard for _provable_ additive rank-error guarantees with near-optimal space. With default K=200, it achieves **~1.65% normalized rank error** in roughly 1.8 KB serialized. It's fully mergeable, order-insensitive, and handles arbitrary comparable types. [Apache DataSketches](https://datasketches.apache.org) provides production-grade KLL in Java, C++, Python, and Rust, integrated into Druid, Hive, Pinot, and BigQuery.

**t-digest** (Dunning, arXiv:1902.04023) excels at tail quantiles (P99, P99.9) with accuracy proportional to max(q, 1−q), achieving "part per million accuracy for extreme quantiles." However, it **lacks formal worst-case guarantees**—rank error can reach ~100% on adversarial blocky data—and is only one-way mergeable. The [reference Java implementation](https://github.com/tdunning/t-digest) is embedded in Elasticsearch, Redis, PostgreSQL, and Apache Druid. Rust, Go, and Python ports exist.

**Moments Sketch** (Gan et al., VLDB 2018) is the most compact at **~200 bytes** (storing min, max, count, and ~15 power sums) with **50 ns merge time**—the fastest merge of any sketch by 15–50×. The tradeoff: quantile estimation requires solving a maximum entropy optimization (millisecond range), and accuracy degrades on heavy-tailed or multimodal data. The newest entrant, **SplineSketch** (SIGMOD 2025), uses monotone cubic spline interpolation to achieve **2–20× better accuracy than t-digest** on non-skewed data with formal near-optimal guarantees and full mergeability.

|Sketch|Insert rate|Space|Error type|Formal bounds|Fully mergeable|
|---|---|---|---|---|---|
|DDSketch|**~100M+/s**|O(log(range)/log(1+α))|Relative value|Yes|**Yes**|
|KLL (K=200)|Millions/s|~1.8 KB|Additive rank ±1.65%|Yes|**Yes**|
|t-digest|Millions/s|~2–10 KB|Relative to max(q,1−q)|No|No (one-way)|
|Moments|Millions/s, **50ns merge**|**~200 bytes**|Average rank|Partial|**Yes**|
|ReqSketch|Millions/s|O(log^1.5(εn)/ε)|Relative rank|Yes|**Yes**|
|GK|~6× slower than DDSketch|O((1/ε)log(εn))|Additive rank|Yes (optimal deterministic)|No|

## Python, Java, Go, and Rust: the practical implementations

**Python** has no built-in equi-depth histogram function, but the standard pattern is fast and well-supported. `np.quantile(data, np.linspace(0, 1, B+1))` computes bin edges via C-optimized sorting, then `np.histogram(data, bins=edges)` assigns counts—this is roughly **10,000× faster** than equivalent pandas operations for large arrays. `pandas.qcut()` wraps this with Categorical labels but adds overhead from interval objects. For ML pipelines, `sklearn.preprocessing.KBinsDiscretizer(strategy='quantile')` provides a fit/transform API and defaults to **subsampling at 200K rows** for scalability. For streaming, `pip install datasketches` gives Python bindings to the C++ DataSketches KLL/REQ implementations, while `pip install ddsketch` provides Datadog's DDSketch.

**Java** is the richest ecosystem. [Apache DataSketches](https://github.com/apache/datasketches-java) offers KLL, Classic Quantiles, REQ, and t-digest with cross-language binary compatibility. The [t-digest reference](https://github.com/tdunning/t-digest) by Ted Dunning is the canonical implementation used by Elasticsearch. Apache Spark's `EquiHeightHistogram` uses the **Greenwald-Khanna algorithm** via `ApproximatePercentile` to build 254-bucket equi-height histograms for its cost-based optimizer. HdrHistogram (Gil Tene) records values at **3–6 ns per insert** with constant memory—not equi-depth by design, but supports arbitrary percentile queries from its logarithmic bucketing structure.

**Rust** implementations center on the crate ecosystem. The `sketches-ddsketch` crate benchmarks at **70M+ inserts/sec**. The `tdigest` crate follows Facebook Folly's implementation. The `kll-rs` crate ports DataSketches' KLL. For exact computation, Rust's standard library `slice::select_nth_unstable()` (introselect) and `slice::sort_unstable()` (pdqsort) are both highly optimized.

**Go** has strong streaming options: [VividCortex/gohistogram](https://github.com/VividCortex/gohistogram) implements Ben-Haim & Yom-Tov streaming histograms, `bmizerany/perks/quantile` provides the CKMS algorithm used by **Prometheus summary metrics**, and multiple t-digest ports ([spenczar/tdigest](https://github.com/spenczar/tdigest) at 1–4 µs per insert, [caio/go-tdigest](https://github.com/caio/go-tdigest)) round out the options.

## How databases build equi-depth histograms internally

**PostgreSQL** uses the most influential design: a hybrid of **MCV (Most Common Values) list + equi-depth histogram** on the remaining values. During `ANALYZE`, it samples 300 × `default_statistics_target` rows (default 30,000) using Vitter's Algorithm Z reservoir sampling, sorts the sample, extracts MCVs, then picks evenly spaced values from the remainder as histogram boundaries. This hybrid prevents double-counting and gives exact estimates for frequent values. The source lives in `src/backend/commands/analyze.c`.

**MySQL 8.0** added explicit histogram support with two types: **singleton** (when NDV ≤ bucket count) and **equi-height** (otherwise). Version 8.0.30 introduced an improved algorithm that prevents skewed values from being packed into single buckets. Construction is manual-only (`ANALYZE TABLE ... UPDATE HISTOGRAM ON col WITH N BUCKETS`), uses up to 20 MB of memory, and falls back to Bernoulli sampling when data exceeds this limit. Buckets store lower bound, upper bound, cumulative frequency, and NDV—up to 1,024 buckets.

**Oracle** evolved through four histogram types, settling on **hybrid histograms** (12c+) that combine frequency tracking for popular values with height-balanced distribution for the rest. Construction uses streaming HyperLogLog for NDV approximation and a lossy counting (Space-Saving) variant for top-N identification—a sophisticated pipeline that runs with `AUTO_SAMPLE_SIZE` doing dynamic sampling. **DuckDB notably eschews histograms entirely**, relying instead on per-row-group min/max zonemaps and HyperLogLog distinct counts, reflecting a modern columnar philosophy where physical data layout enables effective pruning without detailed distribution statistics.

**Apache Spark** constructs equi-height histograms through a **two-scan approach**: first computing percentile endpoints with the Greenwald-Khanna algorithm (`ApproximatePercentile`), then counting distinct values per bucket with HyperLogLog++ (`ApproxCountDistinctForIntervals`). This is disabled by default and requires `spark.sql.statistics.histogram.enabled = true`. **CockroachDB** performs distributed full-table scans with HyperLogLog (HLL-TailCut+) for distinct counts and probabilistic auto-refresh after ~20% row staleness.

## Academic benchmarks and the algorithmic frontier

The most rigorous experimental comparison is Fernando, Bindra & Daudjee's **"An Experimental Analysis of Quantile Sketches over Data Streams" (EDBT 2023)**, which tested KLL, Moments, DDSketch, UDDSketch, and ReqSketch in Apache Flink. Their key finding: **no single algorithm dominates**—DDSketch wins on insertion and query speed, Moments wins on merge speed and compactness, UDDSketch provides the best relative-error guarantees, and ReqSketch excels at upper tail accuracy.

The 2025 **SplineSketch** paper (SIGMOD 2025, arXiv:2504.01206) represents the current frontier, achieving 2–20× better accuracy than t-digest with formal near-optimal guarantees and full mergeability. On the theoretical side, Cormode & Veselý (PODS 2020) proved Greenwald-Khanna is **space-optimal among deterministic comparison-based summaries**, while Gupta, Singhal & Wu (FOCS 2024) explored non-comparison-based algorithms exploiting universe size.

For equi-depth histograms specifically, the **BASH algorithm** (Mousavi & Zaniolo, EDBT 2011) achieves **4× speedup** over GK-based approaches for streaming equi-depth construction over sliding windows. Yıldız & Senkul's merge-based approach (arXiv:1606.05633) targets Hadoop/HDFS distributed construction with quality guarantees.

## Conclusion

The fastest path to equi-depth histograms depends on your constraints. For **maximum throughput on streaming data**, DDSketch at 100M+ inserts/sec with full mergeability is unmatched—use the Rust crate for raw speed or DataDog's official Java/Python/Go libraries. For **exact results on in-memory data**, SIMD-vectorized sorting (VQSort or x86-simd-sort) followed by boundary selection runs at ~1 GB/s on modern CPUs, while the `multi_nth_element` algorithm offers theoretically optimal O(n log B) complexity. For **provable guarantees in production**, Apache DataSketches' KLL sketch is the industry standard across Java, C++, and Python. The newest contender to watch is **SplineSketch (SIGMOD 2025)**, which may redefine the accuracy-space tradeoff. In databases, PostgreSQL's MCV + equi-depth hybrid remains the most widely adopted design, though DuckDB's histogram-free approach signals a shift in columnar engine philosophy.