---
name: skill-router-master
description: Master merged skill for RabiCLI — routes to domain sub-skills (code quality/speed per language, deep debug/test, GitHub top-15 features, hook improvements). Combines all RabiCLI skills + top GitHub OSS agent features into one router.
---

# /rabicli-merged-master — RabiCLI Master Merged Skill

This skill is the single entry point for all RabiCLI upgrades. It reads your task, picks the right domain sub-skill(s), loads only those, and executes. It is the merged skill: **one skill that calls all the other skills.**

## THE AUTONOMY CONTRACT (STANDING — OVERRIDES EVERYTHING ELSE)

1. **PLAN AND EXECUTE IMMEDIATELY.** After writing any plan, START EXECUTING in the same breath. The plan file is a RECORD, not a permission request.
2. **FACTS vs. DECISIONS:** A fact (does this endpoint return 200? what does this function do?) → resolve it yourself: test it in the terminal, read the source, or web_search it. NEVER ask the operator about a testable fact. A decision (which approach, which order) → apply the SAFE DEFAULT, log it, CONTINUE. Never stop to ask.
3. **THE ONLY HARD STOP** is the final deliverable on disk. Every stop for permission before that is a task failure.
4. **DESTRUCTIVE actions** (deleting data beyond task scope, OS-level commands on production systems) get a one-line confirm. Everything else proceeds without asking.

## THE ANTI-STALL RULE

After you read ANY file or write ANY plan, your next action is the NEXT EXECUTION STEP — never a summary of what you just read.

## ROUTER — HOW THE MERGED SKILL PICKS SUB-SKILLS

Read the task, classify into one primary domain + up to 2 overlays, then invoke only those sub-skills:

| If the task mentions... | Primary sub-skill | Overlay options |
|---|---|---|
| Python code, .py, pip, uv, pytest | `py-quality-gate` | `deep-test-all`, `github-feature-mine` |
| C# code, .cs, .sln, .csproj, dotnet | `dotnet-quality-gate` | `deep-test-all`, `github-feature-mine` |
| C++ code, .cpp, .h, cmake, clang | `cpp-quality-gate` | `deep-test-all`, `github-feature-mine` |
| debug, test, verify, prove, reproduce | `deep-test-all` | `py-quality-gate`, `dotnet-quality-gate`, `cpp-quality-gate` |
| GitHub repo, best features, steal ideas, upgrade | `github-feature-mine` | `py-quality-gate`, `dotnet-quality-gate`, `cpp-quality-gate` |
| hook, pre-commit, post-edit gate, safety | `hook-gates` | — |
| parallel agents, spawn subagents, fan-out, orchestrate 5–30 agents | `subagents` | `godmode`, `honestscore` |
| RabiCLI workspace, upgrade, new version | `skill-router-master` (this one) | all sub-skills |

**Invocation rule:** Call the sub-skill directly (not through this router) once the domain is clear. This router exists to classify and delegate — it does not re-implement the sub-skill bodies.

## SUB-SKILL 1 — CODE QUALITY & SPEED (PYTHON)

**When:** Task involves Python code, .py files, pip/uv, pytest, Django, FastAPI, etc.

**What it adds (from top GitHub repos Oct 2026):**

Linters/formatters/type-checkers:
- **Ruff** — ~48,000 stars (MIT). One tool replacing flake8, pylint, isort, pyupgrade, pydocstyle. 10-100x faster. `ruff check`, `ruff format`, `ruff --select E,F,W,C,I,N,UP`
- **mypy** — type checker, `mypy --strict <file>`
- **pyrefly** — Meta, 1.0 released 2026-05-12, fast Python type checker (MIT). ~1.85M LOC/sec
- **ty** — Astral's beta type checker, 0.0.89 (2026-10-16), MIT

Profilers/speed:
- **py-spy** — 15,567 stars. Profiles running Python without restart. `py-spy record -o profile.svg -- python script.py`
- **memray** — 15,378 stars. Tracks every allocation including native extensions. `memray run -o output.bin script.py`
- **scalene** — 13,525 stars. CPU + GPU + memory profiler. `scalene --cli script.py`
- **pyinstrument** — 8,014 stars. Readable call-tree profiler. `pyinstrument script.py`

Testing:
- **Hypothesis** — 9,079 stars. Property-based testing. `pytest` with `@given`
- **mutmut** — 1,474 stars. Mutation testing for Python. `mutmut run`
- **pytest-benchmark** — 1,456 stars. `pytest --benchmark`
- **bandit** — 8,268 stars. Security linting for Python. `bandit -r .`
- **pip-audit** — 1,381 stars. Audits Python env for known vulns. `pip-audit`

**Exit codes:** PASS=0 (all checks green), FAIL=1 (real defect), COULD-NOT-MEASURE=2 (no tests, missing SDK, build failure).

## SUB-SKILL 2 — CODE QUALITY & SPEED (C# / .NET)

**When:** Task involves C# code, .cs files, .sln/.csproj, dotnet CLI, WinForms, ASP.NET Core.

**What it adds:**

Linters/analyzers:
- **Roslyn analyzers** — built into .NET SDK. `dotnet build` with analyzers enabled
- **StyleCop.Analyzers** — C# style and consistency rules
- **Roslynator** — 4,400+ analysis suggestions

Benchmarking:
- **BenchmarkDotNet** — 11,505 stars. The .NET benchmarking standard.

Testing:
- **xUnit / NUnit / MSTest / TUnit** — test frameworks. `dotnet test --filter "FullyQualifiedName~X"`
- **coverlet** + **ReportGenerator** — code coverage. `dotnet test --collect:"XPlat Code Coverage"`
- **Stryker.NET** — 2,098 stars. Mutation testing for .NET. `dotnet stryker --break-at 40`
- **FsCheck** — property-based testing for .NET

Profiling:
- **dotnet-trace** — CPU profiling
- **dotnet-dump** — memory dumps
- **Tracy** — 16,906 stars. Real-time nanosecond-resolution frame profiler

CRAP metric: comp² × (1 − cov)³ + comp (from ReportGenerator Risk Hotspots — NOT Aaronontheweb's wrong formula).

## SUB-SKILL 3 — CODE QUALITY & SPEED (C++)

**When:** Task involves C++ code, .cpp/.h files, CMake, clang, g++.

**What it adds:**

Linters/static analysis:
- **clang-tidy** — C++ linter from LLVM. `clang-tidy -checks=* <file>`
- **cppcheck** — static analysis. `cppcheck --enable=all <dir>`
- **clang-format** — formatter. `clang-format -i <file>`

Benchmarking:
- **Google Benchmark** — 10,469 stars.

Testing:
- **Google Test** — test framework. `gtest` with `EXPECT_*` assertions
- **Catch2** — modern C++ test framework
- **gcov/lcov** — coverage

Profiling:
- **perf** — Linux CPU profiler
- **Tracy** — 16,906 stars. Real-time frame profiler for C++
- **perfetto** — 6,636 stars. System-wide tracing
- **AddressSanitizer** — `-fsanitize=address`
- **ThreadSanitizer** — `-fsanitize=thread`
- **UndefinedBehaviorSanitizer** — `-fsanitize=undefined`

## SUB-SKILL 4 — DEEP DEBUG & TEST (EVERY NEW FUNCTION)

**When:** Task says "debug", "test all functions", "verify", "prove it works", "last step - debug testing".

This is the core debugging/testing skill merged from the TOP-10 repos (obra/superpowers, mattpocock/skills, dotnet/skills, addyosmani/agent-skills, garrytan/gstack, wshobson/agents, trailofbits/skills, anthropics/claude-plugins-official, openai/plugins, github/awesome-copilot).

**Eight phases (run in order, each writes to disk, next reads from disk):**

| Phase | What it does | Exit condition |
|---|---|---|
| 0 Detect & baseline | Detect stack. Run existing suite once from fresh build, record counts | Baseline recorded, or exit 2 |
| 1 Inventory every function | List every function/method. Pair with existing tests | `inventory.jsonl` complete |
| 2 Design cases per function | Happy path, empty, boundary, invalid, error path, concurrency, security | Case list written |
| 3 Write & run, narrow first | Use framework already in solution. Prove every new test can fail | Each test has red-proof |
| 4 Debug each failure | Reproduce gate, classify, root cause, 3-5 falsifiable hypotheses | FIXED with evidence |
| 5 Test-quality gate | Audit for no-assertion, tautological, vacuous, swallowed-exception, sleep-based | Audit table + mutation results |
| 6 Clean final verification | Delete build outputs. Run FULL suite with results file. Branch coverage | Exit 0 with executed > 0, or honest 1/2 |
| 7 Report & gate | Per-function evidence table. Run check_report.py | Gate passes or report says what's missing |

**Hard rules:**
1. Evidence or it did not happen — every pass/fail shows exact command, exit code, counts parsed from output files.
2. Exactly three outcomes: PASS(0), FAIL(1), COULD-NOT-MEASURE(2). A 2 is never reported as a pass.
3. Never weaken a test to get green. No skip, xfail, [Fact(Skip=…)], [Ignore], loosened assertion, or retries.
4. Never pause to ask permission. Record blocked item with evidence, continue with next function.
5. One phase at a time, from files — each phase writes under `.debug-test-all/`, next phase reads from there.

## SUB-SKILL 5 — GITHUB TOP-15 FEATURES

**When:** Task says "steal ideas from GitHub", "upgrade our tool", "best features from top repos".

**Top 15 features from top-tier GitHub repos (Oct 2026):**

From obra/superpowers (300.5k stars, MIT):
1. True red/green TDD loop with whole-suite rule
2. Systematic debugging with evidence collection
3. Root-cause analysis with falsifiable hypotheses

From mattpocock/skills (293.9k stars, MIT):
4. Red-green-refactor TDD loop for agent skills
5. Type-safe skill definitions

From addyosmani/agent-skills (107.3k stars, MIT):
6. Full loop: spec, test, review, security, web-performance audits
7. Agent skill composition patterns

From anthropics/defending-code-reference-harness (7.6k stars, Apache-2.0):
8. Dedupe before verify (deterministic pass + semantic agent)
9. Three independent verifiers per finding, never forked from main context
10. Verdict block: verdict, 0-10 confidence, refute reason, exclusion rule, first call site

From garrytan/gstack (136.7k stars, MIT):
11. Evidence-based debugging with file:line citations
12. Blast-radius analysis

From trailofbits/skills (7.6k stars, CC-BY-SA-4.0 — paraphrase only):
13. Six mandatory gates before calling anything a vulnerability
14. Variant analysis: start from exact match, generalize one element at a time
15. Post-patch validation: baseline vs patched with control checks

**Application rule:** For each feature, report: repo URL, stars, what it does, how to apply it to RabiCLI, and whether MIT/Apache licensed (safe to copy ideas) or CC-BY-SA (paraphrase only).

## SUB-SKILL 6 — RABICLI HOOK IMPROVEMENTS

**When:** Task says "hooks", "pre-commit", "post-edit gate", "safety", "auto-validate".

**Current RabiCLI hook system:**
- JSON control output: `additionalContext`, `systemMessage`, `continue`
- Lifecycle hooks: SessionStart, UserPromptSubmit, PostToolUse
- Permission modes: `smart`, `autonomous-sandbox`

**Improvements to add:**

1. **Post-edit code quality gate:** After any file write/edit, run the domain's fast linter within a time budget.
2. **UserPromptSubmit hook:** Match debug/test trigger phrases ("last step - debug", "debug testing all functions", "test all functions please", "prove it works") and inject the deep-debug-test skill context.
3. **Stop gate:** Block up to twice if the debug-test skill was never invoked on a task mentioning testing. Kill switch: `~/.opencode/skill-gate.off` or `SKILL_GATE_OFF=1`.
4. **Always-on token budget:** Each skill reports its always-on token count; if > threshold, warn before loading.

## EXECUTION PROTOCOL

When you invoke `/rabicli-merged-master <task>`:

1. **Classify:** Read the task, pick primary domain + overlays from the router table above.
2. **Delegate:** Invoke the matching sub-skill(s) directly with the task description.
3. **Execute:** The sub-skill runs its phases, writing every intermediate result to disk under `.rabicli-merged/` or `.debug-test-all/`.
4. **Verify:** Every claim backed by terminal output, source URL, or live test. Exit code 0/1/2 reported honestly.
5. **Report:** Write a mandatory task report to `C:\Users\Administrator\Desktop\reports\<YYYY-MM-DD_HHMMSS>\<short-task-name>.md`

## MANDATORY TASK REPORT

Path: `C:\Users\Administrator\Desktop\reports\<YYYY-MM-DD_HHMMSS>\<short-task-name>.md`

Sections:
1. `# <title>` — name the task
2. `## Task` — what was requested
3. `## What changed` — every file created/edited/moved
4. `## Files` — every path read and written
5. `## Result` — measured outcome, use tables for ids/counts/times/scores
6. `## How to check` — numbered steps to confirm from files/commands
7. `## Not done` — what was left unfinished

## SELF-CHECK

- Every feature from GitHub repos cited with URL and star count?
- Every pass/fail backed by terminal output?
- Every test proven red before green?
- Every sub-skill exit code honest (0/1/2)?
- Report written to disk?

Target honest score: 9.5/10. If below, identify the gap and fix it before declaring done.