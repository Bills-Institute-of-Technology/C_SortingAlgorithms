# C_SortingAlgorithms — Project Specification

**Status:** Draft
**Last updated:** 2026-08-19
**Language standard:** ANSI C (C89/C90), no compiler extensions
**Build system:** CMake (>= 3.15), targeting MSVC, GCC, and Clang

---

## 1. Purpose

This project implements classic sorting algorithms directly in ANSI C, rather than
deferring to a standard library routine such as `qsort()`. It has two goals:

1. **Review** — each algorithm lives in its own source file, written for legibility,
   so the low-level mechanics of each strategy can be read and compared side by side.
2. **Measure** — a single driver program runs any or all of the algorithms against a
   shared set of datasets and reports comparable performance metrics: iteration counts,
   comparison counts, data movement, and elapsed time.

### Non-goals

- Beating the standard library on performance. Instrumentation is deliberately traded
  for measurability; an instrumented `quicksort` will lose to `qsort()`.
- Multithreaded or SIMD sorting.
- A general-purpose sorting library for third-party consumption.

---

## 2. Design principles

| Principle | Consequence |
| --- | --- |
| One algorithm per translation unit | Each algorithm is reviewable in isolation; no shared helper creep. |
| Algorithms are data-type agnostic | Sorting operates on an opaque byte array plus a comparator, in the style of `qsort()`. The same `bubble_sort.c` sorts integers, strings, and structs. |
| Instrumentation is external to the algorithm | Algorithms call small helpers (`sa_cmp`, `sa_swap`, `sa_tick`) that record metrics as a side effect. No algorithm file contains reporting or timing logic. |
| No global mutable state | All state travels in a context struct, so runs are repeatable and independent. |
| Datasets are files, not literals | Test data lives under `data/`, is loaded at runtime, and can be swapped or extended without recompiling. |
| Fair comparison | Every algorithm sees a byte-identical copy of the input for every repetition. |

---

## 3. Directory organization

```
C_SortingAlgorithms/
├── CMakeLists.txt              # top-level build definition
├── README.md
├── LICENSE
├── docs/
│   └── SPECIFICATION.md        # this document
├── include/sorting/            # public headers (installed/interface)
│   ├── sa_types.h              # sa_context, sa_metrics, comparator typedefs
│   ├── sa_algorithm.h          # sa_algorithm descriptor + registry API
│   ├── sa_instrument.h         # sa_cmp / sa_swap / sa_move / sa_tick helpers
│   ├── sa_dataset.h            # dataset loading and description
│   ├── sa_timer.h              # monotonic clock abstraction
│   └── sa_report.h             # report formatting
├── src/
│   ├── main.c                  # CLI entry point — the "master file"
│   ├── registry.c              # table of all available algorithms
│   ├── instrument.c            # metric-recording helpers
│   ├── dataset.c               # dataset parsing/loading, per-type comparators
│   ├── timer.c                 # platform clock shim (only non-portable file)
│   ├── report.c                # table / CSV / JSON output
│   └── algorithms/
│       ├── bubble_sort.c       # Phase 1
│       └── merge_sort.c        # Phase 1
├── data/
│   ├── README.md               # dataset format reference
│   ├── numeric/
│   ├── strings/
│   └── structured/
├── tools/
│   └── gen_dataset.c           # synthetic dataset generator
└── tests/
    ├── test_correctness.c      # every algorithm sorts every dataset correctly
    ├── test_dataset.c          # loader edge cases
    └── test_instrument.c       # counters behave as specified
```

**Rule:** a file under `src/algorithms/` may include only `sa_types.h`,
`sa_algorithm.h`, `sa_instrument.h`, and ANSI C standard headers. It may not include
`sa_report.h`, `sa_timer.h`, or `sa_dataset.h`. This keeps each algorithm file a
self-contained, reviewable unit and is enforced by review, not by tooling.

---

## 4. Core interfaces

### 4.1 Comparator

```c
/* Returns <0, 0, >0 — same contract as the qsort() comparator. */
typedef int (*sa_cmp_fn)(const void *a, const void *b);
```

Comparators are supplied by the dataset layer, one per element type
(`sa_cmp_long`, `sa_cmp_double`, `sa_cmp_cstr`, `sa_cmp_record_by_key`, ...).

### 4.2 Metrics

```c
typedef struct {
    unsigned long comparisons;      /* sa_cmp calls */
    unsigned long swaps;            /* sa_swap calls */
    unsigned long moves;            /* element-sized copies (sa_move) */
    unsigned long loop_iterations;  /* sa_tick calls — one per inner-loop body */
    unsigned long outer_passes;     /* sa_pass calls — one per outer pass/partition */
    unsigned long max_depth;        /* peak recursion depth */
    unsigned long allocations;      /* scratch allocations requested */
    size_t        peak_extra_bytes; /* peak auxiliary memory in use */
} sa_metrics;
```

Counters are `unsigned long` (guaranteed >= 32 bits in C89). The report layer flags a
run whose counters could plausibly have wrapped so a silent overflow is never
presented as a real measurement.

### 4.3 Sort context

```c
typedef struct {
    void        *base;        /* contiguous array of count elements */
    size_t       count;
    size_t       elem_size;
    sa_cmp_fn    compare;
    sa_metrics   metrics;     /* zeroed before each run */
    void       *scratch_pool; /* opaque; used by sa_alloc_scratch */
} sa_context;
```

### 4.4 Instrumentation helpers

Declared in `sa_instrument.h`, defined in `instrument.c`:

```c
int    sa_cmp(sa_context *ctx, const void *a, const void *b);
void   sa_swap(sa_context *ctx, void *a, void *b);
void   sa_move(sa_context *ctx, void *dst, const void *src);
void   sa_tick(sa_context *ctx);                 /* inner-loop iteration */
void   sa_pass(sa_context *ctx);                 /* outer pass / partition */
void   sa_enter(sa_context *ctx, unsigned long depth); /* recursion depth probe */
void  *sa_alloc_scratch(sa_context *ctx, size_t bytes);
```

Every algorithm must route all element access through these helpers so counts are
directly comparable between algorithms. In particular:

- `sa_tick` is called exactly once per execution of the innermost loop body.
- `sa_pass` is called once per outer-loop pass (bubble) or once per partition/merge
  invocation (quick, merge).
- An algorithm that reads an element without comparing it still calls neither
  `sa_cmp` nor `sa_move`; only real comparisons and real copies are counted.

### 4.5 Algorithm descriptor

```c
typedef void (*sa_sort_fn)(sa_context *ctx);

typedef struct {
    const char *name;          /* CLI selector, e.g. "bubble" */
    const char *display_name;  /* "Bubble Sort" */
    const char *complexity_best;
    const char *complexity_avg;
    const char *complexity_worst;
    const char *space;         /* auxiliary space, e.g. "O(1)" / "O(n)" */
    int         stable;        /* 1 = stable, 0 = not */
    int         in_place;
    sa_sort_fn  sort;
} sa_algorithm;
```

Each algorithm file defines exactly one exported symbol:

```c
/* src/algorithms/bubble_sort.c */
const sa_algorithm sa_algo_bubble = { "bubble", "Bubble Sort",
    "O(n)", "O(n^2)", "O(n^2)", "O(1)", 1, 1, bubble_sort };
```

`registry.c` holds the single array of pointers to these descriptors. **Adding an
algorithm means adding one source file, one `extern` declaration, one array entry,
and one line in `CMakeLists.txt` — nothing else.**

---

## 5. Datasets

### 5.1 Location and naming

Datasets live under `data/<category>/<name>.dat`, where category is `numeric`,
`strings`, or `structured`. The name encodes shape and size, e.g.
`random_10k.dat`, `sorted_10k.dat`, `reversed_10k.dat`, `few_unique_10k.dat`.

Committed fixtures stay small (<= 10,000 records) so the repository remains
lightweight. Larger inputs are produced on demand by `tools/gen_dataset` and are
git-ignored.

### 5.2 File format

A dataset is a plain text file: a metadata header of `#`-prefixed lines, then one
record per line. The header is parsed with `fgets` + `sscanf`; no third-party parser
is required.

```
# sa-dataset: 1
# type: i32 | f64 | string | record
# count: 10000
# key: <field name>          (record only)
# fields: id:i32|name:string|score:f64   (record only)
# delimiter: |               (record only, default '|')
# shape: random | sorted | reversed | few-unique | nearly-sorted
# note: free text
<record 1>
<record 2>
...
```

Rules:

- `sa-dataset`, `type`, and `count` are required; the loader fails with a diagnostic
  if any is missing, if `count` disagrees with the number of records read, or if a
  record fails to parse.
- Header keys are case-insensitive; unknown keys are ignored with a warning so the
  format can grow without breaking older loaders.
- Lines that are empty or begin with `#` after the header block are skipped.
- Records are `\n`-terminated; `\r\n` is tolerated.
- Strings are raw UTF-8 bytes, not quoted or escaped; a string record therefore
  cannot contain a newline. Comparison is bytewise (`strcmp`), not locale-aware.

### 5.3 Type mapping

| `type` | In-memory element | Default comparator |
| --- | --- | --- |
| `i32` | `long` | `sa_cmp_long` |
| `f64` | `double` | `sa_cmp_double` |
| `string` | `char *` into one owned block | `sa_cmp_cstr` |
| `record` | fixed-size struct, fields per header | `sa_cmp_record_by_key` |

For `string`, all text is read into a single allocation and the sorted array holds
pointers into it. Sorting therefore moves pointers, not characters — this is called
out in the report, because it makes string runs cheaper per `sa_move` than numeric
runs and the two are not directly comparable.

For `record`, the loader builds a fixed-size struct from the declared fields and
compares on the field named by `key`.

### 5.4 Required starter datasets

Each category ships at least: `random`, `sorted`, `reversed`, and `few_unique` at
1,000 and 10,000 records. The `sorted` and `reversed` shapes exist specifically to
exercise best/worst-case behavior — bubble sort's early-exit optimization should be
visible in the metrics.

---

## 6. The driver (`src/main.c`)

### 6.1 Command line

```
sorter [options]

  --list                      List available algorithms and datasets, then exit.
  --algo NAME[,NAME...]       Algorithms to run (default: all).
  --data PATH[,PATH...]       Dataset files to run (default: all under data/).
  --category CAT[,CAT...]     Restrict to numeric | strings | structured.
  --repeat N                  Timed repetitions per pair (default: 5).
  --warmup N                  Untimed warmup repetitions (default: 1).
  --max-n N                   Truncate each dataset to its first N records.
  --report FORMAT             table (default) | csv | json.
  --out PATH                  Write report to PATH instead of stdout.
  --verify                    Check sortedness (and stability) after each run.
  --no-verify                 Skip verification.
  --quiet                     Suppress progress output.
  --help                      Usage.
  --version
```

Unknown options are an error, not a warning. Selectors that match nothing are an
error, and the message lists valid values.

### 6.2 Execution model

For each (algorithm, dataset) pair:

1. Load the dataset once; keep a pristine master copy.
2. Run `--warmup` untimed repetitions to page in memory and warm caches.
3. For each of `--repeat` repetitions:
   a. `memcpy` the master copy into the working buffer.
   b. Zero `ctx.metrics`.
   c. Read the clock, call `algo->sort(&ctx)`, read the clock.
   d. If verifying, confirm the result is non-decreasing under the comparator; for
      an algorithm declaring `stable = 1`, confirm equal keys retain their original
      relative order using an index tag attached during loading.
   e. Record the repetition's metrics and elapsed time.
4. Report the **median** elapsed time as the headline figure, with min and max
   alongside. Median resists the scheduling outliers that make means misleading on
   a desktop OS.

Counters are asserted identical across repetitions of a deterministic algorithm; a
mismatch is reported as a defect, since it means the algorithm read uninitialized
state or depended on address values.

Verification runs *outside* the timed region and its comparisons are not counted.

### 6.3 Reporting

`table` format, one row per (algorithm, dataset):

```
Dataset: numeric/random_10k.dat   type=i32  n=10000  repeats=5

Algorithm        Compares      Moves      Inner     Passes   Median ms   ns/iter   Verify
---------------  ----------  ---------  ---------  ---------  ---------  --------  ------
Bubble Sort      49,995,000  74,952,318  49,995,000     9,999    112.480      2.25  ok
Merge Sort          120,433     267,232    143,616        14      1.212      8.44  ok
```

- **Inner** is `loop_iterations` — the "number of loops required" figure.
- **ns/iter** is median elapsed time divided by `loop_iterations`: the average cost
  of one loop iteration, which is the per-loop timing figure. It is the fairest
  cross-algorithm number, since it normalizes away the fact that an O(n²) algorithm
  simply executes more iterations.
- `csv` and `json` formats carry the same fields plus every repetition's raw
  measurements, for downstream plotting.

A trailing summary section ranks algorithms per dataset by median time and notes any
verification failure. Exit status is `0` if every run sorted correctly, `1` on a
verification failure, `2` on a usage or I/O error.

### 6.4 Timing

`timer.c` is the only file permitted to use platform-specific APIs, selected by
CMake:

| Platform | Source |
| --- | --- |
| Windows | `QueryPerformanceCounter` |
| POSIX | `clock_gettime(CLOCK_MONOTONIC, ...)` |
| Fallback | ANSI `clock()` from `<time.h>` |

The interface is `double sa_now_ms(void)` plus `double sa_timer_resolution_ms(void)`.
The ANSI fallback has coarse resolution (often ~10 ms), so the driver reports the
detected resolution in the report header and warns when a measured duration is under
20x that resolution — a fast merge sort on 1,000 elements is otherwise pure noise.
Algorithm files never see any of this.

---

## 7. Build

```
cmake -S . -B build
cmake --build build
./build/sorter --list
```

- `CMAKE_C_STANDARD 90`, `CMAKE_C_EXTENSIONS OFF`.
- Warnings: `-Wall -Wextra -pedantic -Werror` on GCC/Clang, `/W4 /WX` on MSVC.
  MSVC additionally needs `_CRT_SECURE_NO_WARNINGS` for `fopen`/`sscanf`.
- Targets: `sorting` (static library of the algorithms + harness), `sorter`
  (the CLI, links `sorting`), `gen_dataset` (tool), and the `tests/` executables
  registered with CTest.
- Release builds use the toolchain default optimization (`-O2` / `/O2`). Because
  optimization level materially changes results, the report header records the
  compiler, version, and build type.

---

## 8. Phasing

**Phase 1 — harness plus two algorithms (initial scope)**
Types, instrumentation, dataset loader for all three categories, timer, report,
CLI, `bubble_sort.c`, `merge_sort.c`, correctness tests, starter datasets, and the
`gen_dataset` tool. Bubble and merge are chosen deliberately: one O(n²) in-place
and one O(n log n) out-of-place, which together exercise every part of the harness
(`sa_swap` vs `sa_move`, `sa_alloc_scratch`, recursion depth, stability checking).

**Phase 2 — remaining core comparison sorts**
Insertion, selection, shell, quick (with pivot strategy noted in the descriptor),
heap. No harness changes expected; if a harness change *is* needed, that is a signal
the Phase 1 interfaces were wrong and should be revised rather than worked around.

**Phase 3 — non-comparison sorts**
Counting, radix, bucket. These need a key-extraction hook rather than a comparator,
so the descriptor gains an optional `sa_key_fn` and they are restricted to numeric
and `record` datasets.

**Phase 4 — analysis**
`qsort()` baseline row for reference, size-sweep mode (`--sweep 1k,10k,100k`) to plot
growth curves, and CSV output consumed by a small plotting script.

---

## 9. Open questions

1. Should the stability check be mandatory rather than opt-in? It costs an extra
   `unsigned long` tag per element, which perturbs cache behavior for numeric runs.
2. Should `--repeat` default higher (e.g. 11) to make the median more robust, at the
   cost of slow O(n²) runs on 10k inputs?
3. For `record` datasets, is a fixed field schema per file sufficient, or should the
   loader support optional/missing fields?
4. Is a `--csv-append` mode wanted so results accumulate into one file across runs
   for historical comparison?
