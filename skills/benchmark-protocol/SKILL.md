---
id: benchmark-protocol
name: Benchmark Protocol — Before/After Methodology
description: |
  Before/after benchmark methodology for fleet-forge adapters. Covers go test -bench,
  hyperfine for wall-clock, pprof for profiling, and benchstat for comparison.
  Every perf-tuner change must produce a filled benchmark report table.
version: "1.0.0"
category: build-crew/shared
---

# Benchmark Protocol

## Rule Zero

**Measure before you change. Measure after you change. No exceptions.**

A change without a before/after table is an untested change. perf-tuner will not accept
it. forge-architect will not merge it.

---

## Tool Stack

| Tool | Purpose | Install |
|------|---------|---------|
| `go test -bench` | Micro-benchmarks for Go adapter internals | stdlib |
| `benchstat` | Statistical comparison of benchmark runs | `go install golang.org/x/perf/cmd/benchstat@latest` |
| `hyperfine` | Wall-clock CLI benchmarks (subprocess invocation) | `apt install hyperfine` / `cargo install hyperfine` |
| `pprof` | CPU + memory profiling for Go | stdlib (`go tool pprof`) |
| `go-torch` / `speedscope` | Flame graph visualization | optional |

---

## Phase 1: Establish Baseline

Run BEFORE any optimization. Save to a named file.

```bash
# Go micro-benchmark — 5 runs for statistical stability
go test \
  -bench=BenchmarkAdapterInvoke \
  -benchmem \
  -count=5 \
  -timeout=10m \
  ./adapter/<name>/... \
  | tee benchmarks/baseline-<adapter>-$(date +%Y%m%d).txt

# Example output line:
# BenchmarkAdapterInvoke-8   1000   1234567 ns/op   4096 B/op   32 allocs/op
```

```bash
# Wall-clock CLI benchmark with hyperfine
hyperfine \
  --warmup 3 \
  --runs 20 \
  --export-markdown benchmarks/hyperfine-baseline-<adapter>-$(date +%Y%m%d).md \
  'megapp invoke <tool> <args>'
```

---

## Phase 2: Profile to Find the Hot Path

Never guess what to optimize. Profile first.

```bash
# CPU profile
go test \
  -bench=BenchmarkAdapterInvoke \
  -benchtime=10s \
  -cpuprofile=profiles/cpu-<adapter>-<date>.prof \
  ./adapter/<name>/...

go tool pprof -top profiles/cpu-<adapter>-<date>.prof
# Look for functions with >5% cumulative CPU — those are candidates

# Memory profile
go test \
  -bench=BenchmarkAdapterInvoke \
  -benchtime=10s \
  -memprofile=profiles/mem-<adapter>-<date>.prof \
  ./adapter/<name>/...

go tool pprof -top profiles/mem-<adapter>-<date>.prof
# Look for excessive allocs/op in Parse() — common hot spot
```

### Flame Graph (optional but recommended for complex adapters)

```bash
go tool pprof -http=:6060 profiles/cpu-<adapter>-<date>.prof
# Open http://localhost:6060 → Flame Graph view
```

---

## Phase 3: Implement the Optimization

Make the smallest correct change that addresses the profiled hot path.
Do not refactor code that is not in the hot path.

---

## Phase 4: Measure After

```bash
# Run the same benchmark as Phase 1
go test \
  -bench=BenchmarkAdapterInvoke \
  -benchmem \
  -count=5 \
  -timeout=10m \
  ./adapter/<name>/... \
  | tee benchmarks/after-<adapter>-$(date +%Y%m%d).txt

# Compare with benchstat
benchstat \
  benchmarks/baseline-<adapter>-<date>.txt \
  benchmarks/after-<adapter>-<date>.txt
```

Example benchstat output:
```
name              old time/op    new time/op    delta
AdapterInvoke-8    1.23ms ± 2%    0.89ms ± 1%  -27.64%  (p=0.008 n=5+5)

name              old alloc/op   new alloc/op   delta
AdapterInvoke-8    4.10kB ± 0%    1.20kB ± 0%  -70.73%  (p=0.008 n=5+5)
```

A delta is only meaningful if `p < 0.05` (statistically significant). benchstat reports
this automatically.

---

## Phase 5: Sustained Load Verification (resource governor check)

After micro-benchmarks pass, verify the ≤90% CPU/RAM contract under sustained load.

```bash
# Linux
go test -bench=BenchmarkAdapterSustained -benchtime=60s ./adapter/<name>/... &
BENCH_PID=$!
echo "Sampling CPU/RAM for 30s..."
for i in $(seq 1 6); do
    ps -p $BENCH_PID -o %cpu,rss,comm --no-headers 2>/dev/null || echo "process ended"
    sleep 5
done
wait $BENCH_PID

# Expected: %CPU column stays ≤90% in every sample
# Expected: RSS column stays ≤ adapter cap (see resource-governor skill)
```

```powershell
# Windows PowerShell equivalent
$job = Start-Job { go test -bench=BenchmarkAdapterSustained -benchtime=60s ./adapter/<name>/... }
1..6 | ForEach-Object {
    Start-Sleep 5
    Get-Process -Name "go" -ErrorAction SilentlyContinue |
        Select-Object CPU, WorkingSet64, Name |
        Format-Table
}
```

---

## Benchmark Report Template

Fill this template. Every field is required. Write it to `benchmarks/<adapter>-<date>.md`.

```markdown
# Benchmark Report — <adapter-name>
**Date:** <YYYY-MM-DD>
**Phase:** 1 | 2 | 3
**Measured by:** perf-tuner

## Environment
- OS: <linux/windows/darwin>
- CPU: <model, N cores>
- RAM: <GB>
- Go version: <go version output>
- Binary under test: <name + version>

## Baseline (before any change)
| Metric | Value |
|--------|-------|
| p50 latency | Xms |
| p99 latency | Xms |
| allocs/op | N |
| B/op | N bytes |
| Peak CPU (30s load) | X% |
| Peak RAM (30s load) | X MB |

## After Optimization
| Metric | Value | Delta vs Baseline |
|--------|-------|-------------------|
| p50 latency | Xms | -X% |
| p99 latency | Xms | -X% |
| allocs/op | N | -X% |
| B/op | N bytes | -X% |
| Peak CPU (30s load) | X% | — |
| Peak RAM (30s load) | X MB | — |

## Governor Contract
| Check | Result |
|-------|--------|
| TestGovernorKill | PASS / FAIL |
| Sustained CPU ≤90% | PASS (X%) / FAIL (X%) |
| Sustained RAM ≤cap | PASS (XMB / cap YMB) / FAIL |

## Changes Made
- <description of change 1>
- <description of change 2>

## Profiler Evidence
Hot path before: `<function>` at X% CPU / Y allocs/op
Hot path after:  `<function>` at X% CPU / Y allocs/op

## Verdict
PASS | FAIL (reason)

## Unblocks
forge-architect / changelog entry for adapter/<name>
```

---

## Phase 1 Gate — `benchmarks/phase1.md`

Before Phase 2 adapters begin, perf-tuner writes `benchmarks/phase1.md`:

```markdown
# Phase 1 Benchmark Aggregate — 10 Adapters
**Written by:** perf-tuner
**Date:** <YYYY-MM-DD>

| Adapter | p50 (ms) | p99 (ms) | allocs/op | Peak CPU | Peak RAM | GovernorKill | Verdict |
|---------|----------|----------|-----------|----------|----------|--------------|---------|
| nmap | X | X | N | X% | XMB | PASS | PASS |
| sqlmap | X | X | N | X% | XMB | PASS | PASS |
| semgrep | X | X | N | X% | XMB | PASS | PASS |
| ... (10 total) | | | | | | | |

## Phase 1 Gate: PASS (all 10 adapters PASS) | FAIL (N adapters FAIL — list)
```

This file is a hard pre-condition before Phase 2 work begins.

---

## Checklist — Benchmark Turn Complete

- [ ] Baseline captured (`benchmarks/baseline-<adapter>-<date>.txt`)
- [ ] pprof CPU profile captured and hot path identified
- [ ] Optimization implemented (minimal, targeted)
- [ ] After-benchmark captured (`benchmarks/after-<adapter>-<date>.txt`)
- [ ] `benchstat` comparison run and delta is statistically significant (p < 0.05)
- [ ] Sustained 30s load test run; CPU ≤90% and RAM ≤cap verified
- [ ] `TestGovernorKill` passes
- [ ] Benchmark report written to `benchmarks/<adapter>-<date>.md`
- [ ] For Phase 1: `benchmarks/phase1.md` written with all 10 adapters