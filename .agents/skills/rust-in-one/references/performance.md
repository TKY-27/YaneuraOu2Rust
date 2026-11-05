# Performance work

Read this before benchmarking, optimizing, or claiming anything about speed or memory.

## Measure correctly

- Use the shipped optimized profile, usually `--release`. Default development builds are not a useful proxy for release performance; custom profiles and overflow/panic settings also matter.
- Benchmark representative workloads with realistic input sizes and distributions, on the target machine and with the build settings that matter, and state whether the file cache is cold or warm. Confirm correctness first, since a fast wrong answer is not a result.
- Whole-program comparisons: use `hyperfine` if installed, otherwise repeat runs and report median and spread rather than the best run. Report peak memory with `/usr/bin/time -v` (maximum resident set size) where available.
- Micro-benchmarks: use `criterion` (stable) or a justified small timing harness. Use `std::hint::black_box` on inputs and outputs to discourage optimization of the measured work; it is a best-effort hint, not a guarantee.
- Comparing against another implementation: use the same inputs and machine, build both at their shipped optimization levels, verify results against the same correctness contract, and run each multiple times. For statistical claims, decide the acceptable margin before looking at results. Report inconclusive results as inconclusive.

## Find the actual cost

- Locate the cost with a profiler or a controlled experiment before optimizing. Where available, use `perf record --call-graph dwarf` then `perf report`, or `cargo flamegraph` if installed. Keep symbols with a profile such as `[profile.profiling]` with `inherits = "release"` and `debug = true`. For heap usage, use `heaptrack` or the `dhat` crate.
- Reason about growth before timing: bounded numeric values or item counts do not cap bytes per token or line, decompressed bytes, queued work, or retained results. Identify what can remain live simultaneously and which limits the contract actually authorizes.
- Prefer incremental parsing and bounded live state when semantics permit. For a bounded numeric field whose grammar permits leading zeros, incremental accumulation can accept long zero prefixes with constant extra per-token state; collecting the whole token grows with encoded length. Reusing a buffer reduces repeated allocation but does not bound its growth. Preserve accepted input and output/failure order when changing the strategy.
- Costs commonly found in agent-written Rust:
  - unbuffered I/O: `File` without `BufReader`, `println!` per line instead of a locked `BufWriter`;
  - allocating per item: a new `String` per line from `lines()` (reuse one buffer with `read_line`/`read_until` and `clear()` when retaining a whole line is acceptable), `format!`/`to_string()` in loops, collecting intermediate `Vec`s that could stay iterators, `clone()` of large values;
  - hashing: the standard map's default hasher prioritizes collision-attack resistance. Hashing may dominate small-key workloads. Evaluate alternatives against both measurements and the threat model; faster hashers have different security properties, and trusted-key benchmarks do not justify exposing a collision-prone hasher to adversarial inputs;
  - `chars()` or `String` processing on data that is effectively ASCII, where bytes suffice;
  - compiling a `Regex` inside a loop instead of once;
  - lock contention on a shared accumulator in parallel code, instead of per-thread results merged at the end;
  - missing `Vec::with_capacity` or `reserve` when the final size is known.

## Optimize in order

1. Algorithm and data layout: complexity, contiguous `Vec`s over pointer-heavy structures, fewer passes over the data.
2. Allocation and I/O: buffer reuse, streaming, avoiding copies, borrowing from the input buffer.
3. Parallelism (`rayon`), after the single-threaded path is efficient. Check that the speedup scales and that output order stays deterministic.
4. Build settings, each measured: `lto = "thin"` or `"fat"`, `codegen-units = 1`. `panic = "abort"` changes semantics (no unwinding, no `catch_unwind`), so use it only if acceptable. `-C target-cpu=native` makes binaries non-portable; never make it the default for distributed artifacts.
5. Unsafe code (`get_unchecked`, manual SIMD), only with profile evidence, a benchmark showing the gain, and the obligations in [unsafe-ffi.md](unsafe-ffi.md). Bounds checks can often be removed safely by iterating, slicing once (`let s = &v[..n];`), or using `chunks_exact`.

Keep each performance optimization only if relevant measurements show it helps while affected correctness checks still pass. Record the measurement in the report or commit message, not in code comments.
