---
name: dotnet-quality-gate
description: C# / .NET code quality and speed optimization skill — analyzers, formatters, benchmarking, testing, coverage, mutation testing. Part of the rabicli-merged-master skill system.
---

# /rabicli-csharp — C# / .NET Code Quality & Speed Optimization

This skill optimizes C#/.NET code quality and speed using top GitHub tools from the Oct 2026 code-quality & speed report.

## INSTALLATION

```bash
dotnet tool install --global dotnet-format
dotnet tool install --global dotnet-stryker
dotnet tool install --global dotnet-reportgenerator
dotnet add package StyleCop.Analyzers
dotnet add package Roslynator.Analyzers
dotnet add package coverlet.collector
dotnet add package FsCheck
```

## PHASES

### Phase 1 — Build & analyze

```bash
dotnet build <project.csproj>
dotnet build <project.csproj> -c Release
dotnet format --check <project.csproj>
dotnet format <project.csproj>
```

### Phase 2 — Test

```bash
dotnet test <project.csproj>
dotnet test <project.csproj> --filter "FullyQualifiedName~X"
dotnet test <project.csproj> --list-tests
dotnet test <project.csproj> --blame-hang --blame-hang-timeout 5min
```

### Phase 3 — Coverage

```bash
dotnet test --collect:"XPlat Code Coverage"
dotnet test --collect:"XPlat Code Coverage" --logger "trx;LogFileName=results.trx"
reportgenerator -reports:coverage.cobertura.xml -targetdir:coveragereport
```

### Phase 4 — Benchmarking (BenchmarkDotNet)

```csharp
[Benchmark]
public void MyMethod() { ... }
BenchmarkRunner.Run<MyBenchmarks>()
```

```bash
dotnet run -c Release
```

### Phase 5 — Mutation testing (Stryker.NET)

```bash
dotnet stryker
dotnet stryker --break-at 40
dotnet stryker --since:<commit>
```

### Phase 6 — Performance profiling

```bash
dotnet-trace collect --process-id <pid>
dotnet-dump collect --process-id <pid>
```

## CRAP METRIC

```
CRAP = comp² × (1 − cov)³ + comp
```

Read from ReportGenerator Risk Hotspots. Do not use Aaronontheweb's wrong formula.

## EXIT CODES

| Code | Meaning |
|------|---------|
| 0 | PASS — all checks green |
| 1 | FAIL — real defect (build failure, test failure, coverage below threshold) |
| 2 | COULD-NOT-MEASURE — zero tests collected, missing SDK, build failure, timeout |

## RULES

1. Evidence or it did not happen — exact command, exit code, counts from TRX/coverage files.
2. Never weaken a test (no [Fact(Skip=…)], [Ignore], loosened assertions, retries).
3. Fresh build, never `--no-build` for final verification.
4. Report to `C:\Users\Administrator\Desktop\reports\<timestamp>\<task-name>.md`

## EXAMPLE

```
/rabicli-csharp all C:\Users\Administrator\Desktop\Projects\FastSqliTool\FastSqliTool.csproj
```