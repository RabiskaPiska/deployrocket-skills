---
name: cpp-quality-gate
description: C++ code quality and speed optimization skill — clang-tidy, cppcheck, clang-format, Google Benchmark, sanitizers, Google Test, profiling. Part of the rabicli-merged-master skill system.
---

# /rabicli-cpp — C++ Code Quality & Speed Optimization

This skill optimizes C++ code quality and speed using top GitHub tools from the Oct 2026 code-quality & speed report.

## INSTALLATION (run once per machine)

```bash
# LLVM/Clang toolchain (clang-tidy, clang-format, clang++, lld)
sudo apt install clang-tools llvm  # or download from llvm.org

# CMake (build system)
# https://cmake.org/download/

# Google Test / Google Benchmark
git clone https://github.com/google/benchmark.git
git clone https://github.com/google/googletest.git
# or: sudo apt install libbenchmark-dev libgtest-dev
```

## PHASES

### Phase 1 — Lint & static analysis

```bash
clang-tidy -checks='*' -p build <file.cpp>              # comprehensive lint
clang-tidy -checks='clang-analyzer-*,cppcoreguidelines-*,misc-*,performance-*' <file.cpp>
# run-clang-tidy.py runs those checks in parallel over the compilation database; clang-tidy-diff.py reports only the lines a diff touched.
cppcheck --enable=all --inconclusive --std=c++20 <dir>   # static analysis
clang-format --dry-run --Werror --fail-on-incomplete-format <file.cpp>  # format gate
clang-format -i <file.cpp>                                # auto-format
```

### Phase 2 — Build with warnings

```bash
g++ -Wall -Wextra -Wpedantic -Werror -O2 <file.cpp> -o <output>
clang++ -Wall -Wextra -Wpedantic -Werror -O2 <file.cpp> -o <output>
```

### Phase 3 — Test (Google Test / Catch2)

```bash
./<test_binary>                              # run all tests
./<test_binary> --gtest_filter=*TestName*    # narrow run
./<test_binary> --gtest_list_tests           # list all tests
```

### Phase 4 — Benchmarking (Google Benchmark)

```cpp
BENCHMARK(BM_MyFunc);
BENCHMARK_MAIN();
```

```bash
./<benchmark_binary> --benchmark_format=json --benchmark_out=results.json
```

### Phase 5 — Sanitizers (memory/thread/UB)

```bash
g++ -fsanitize=address,undefined -fno-omit-frame-pointer -g -O1 <file.cpp> -o <output>   # ASan + UBSan
g++ -fsanitize=thread -g -O1 <file.cpp> -o <output>   # TSan
```

### Phase 6 — Profiling

```bash
perf record -g ./<program>       # CPU profiling
perf report

# Tracy (17.0k stars) — real-time frame profiler
# Integrate TracyClient into your C++ app, then open .tracy file in Tracy profiler

# perfetto (6.7k stars) — system-wide tracing
# perfetto --txt trace.perfetto-trace
```

### Phase 7 — Coverage (gcov/lcov / llvm-cov)

```bash
g++ --coverage -o <program> <file.cpp>
./<program>                      # run to generate .gcda files
lcov --capture --directory . --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.filtered.info
genhtml coverage.filtered.info --output-directory covreport
# or LLVM:
llvm-cov show ./<program> -instr-profile=default.profdata
```

## EXIT CODES

| Code | Meaning |
|------|---------|
| 0 | PASS — all checks green, all tests passed |
| 1 | FAIL — real defect (lint errors, test failures, sanitizer violations) |
| 2 | COULD-NOT-MEASURE — no tests, missing compiler, build failure, timeout |

## RULES

1. Evidence or it did not happen — exact command, exit code, counts from test output/coverage files.
2. Never weaken a test or suppress a sanitizer finding without documented justification.
3. Compile with `-Wall -Wextra -Wpedantic -Werror` — warnings are errors.
4. Sanitizer violations are always FAIL (1), never downgraded.
5. Report to `C:\Users\Administrator\Desktop\reports\<timestamp>\<task-name>.md`

## EXAMPLE

```
/rabicli-cpp all C:\Users\Administrator\Desktop\Projects\MyCppProject\src\
```