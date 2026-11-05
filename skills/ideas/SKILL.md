---
name: ideas
description: For any found problem/defect/root-cause, generate MANY distinct solutions, score them, test the best, and pick by evidence — instead of fixing with the first idea. Has a code-performance mode (profile first, statistical benchmark kill-criteria, 9.7 bar for small code changes). Partner to /honestscore.
---

# ideas — Divergent→Convergent Problem-Solving

You are running the **Ideas Skill**. A problem has been found. Do NOT jump to the first fix.

**Entry + ship-bar.** If `/honestscore` handed you typed tickets (`defect | point-cost | evidence-that-would-close-it`), work the **highest point-cost first**. Ship-bars: **≥9.0** sites and brain docs · **≥9.5** critical code · **≥9.7** small, specific code changes worked in **code-performance mode**. Default flow is Steps 0→7, but **Step 1 routes you**.

## 0 · Preflight the mistake-memory

Grep brain runbooks / root-cause libraries for the symptom. A hit = candidate #1 (known fix) or a banned option (already failed).

## 1 · Reversibility gate — sets the rigor and the route

- **Two-way door** (cheap to undo — most CSS/config/copy defects): root cause in one line, cheapest sane fix (~70% decision-confidence), apply, verify — Steps 0 → 1 → 6-verify only.
- **One-way door** (migration, deletion, schema, anything expensive to undo) *or unsure*: full procedure, N = 5 to 8–10.
- **Code-performance route** — the defect is speed, latency, throughput, memory or concurrency in code: full procedure, N ≥ 5, and the root cause MUST come from a **profile**, never from reading code: Python `py-spy`/`cProfile`/`scalene`, .NET `dotnet-trace`/`dotnet-counters`, C++ `perf` or Google Benchmark counters, SQL `EXPLAIN (ANALYZE, BUFFERS)`. No candidate list for unprofiled code: measure, hypothesise, change ONE thing, re-measure.

## 2 · Root cause, not symptom

State **2–3 competing root-cause hypotheses, each with a rough confidence**. Run the cheapest test that SEPARATES them before ideating. Code-performance: the hypotheses name the hot frame/plan node from the profile and what share of time it owns; rule out an external bottleneck (DB, network, lock wait) before tuning CPU.

## 3 · Diverge — N distinct solutions

On a one-way door or a code-performance change, generate the N ideas and the critic in separate calls that cannot see each other, so the first idea does not anchor the rest; the two-way-door short path stays in one pass.

Span: quick-fix · proper-fix · reframe · prevent-recurrence · inversion (what to *remove / disable / stop doing* so the defect is impossible) · borrow-from-a-named-reference. **Code-performance adds:** lower-complexity algorithm · better data structure · batch/vectorise · remove work (skip; cache with an explicit invalidation rule; compute lazily) · move work off the hot path · change the concurrency model · fewer allocations · I/O shape · build/runtime flags. **At least two candidates from different axes; no two ideas identical in approach.**

## 4 · Score in one table

`idea | fixes-root-cause | effort | risk | 2026-fit | confidence | kill-criterion`

- fixes-root-cause none/partial/full → **0 / 0.5 / 1** · effort · risk · 2026-fit = low/med/high · confidence (evidence tier) = measured **0.8** / directional **0.5** / hunch **0.25**.
- **Rank** by *fixes-root-cause × confidence*, then higher 2026-fit, then **lower risk, then lower effort**.
- **Kill-criterion** = the observable that proves the idea wrong. Code-performance kill-criteria are statistical, paired with "and the full test suite stays green."
- Two ideas sharing a kill-criterion are one bet — drop one, add an independent alternative. Shortlist the top 1–2.

## 5 · Test — cheapest discriminating test first

Rank the shortlist by test-cost; run the cheapest test that SEPARATES them. Evidence, not reasoning. Code-performance: same harness for every candidate, runs interleaved; keep an **experiment log**, one row per candidate tried. Stop when a dominant winner can't be dethroned by the cheapest remaining test, or the ship-bar is met.

## 6 · Converge + pre-mortem

Pick by measured result. Before applying: "it shipped and failed / regressed — most likely why?" — if plausible, add it as a verification check. A speedup that turns ANY test red is reverted; a delta inside noise is reverted. Apply, then re-score with `/honestscore`. If the real score didn't rise, try the next shortlisted idea or re-frame the root cause.

## 7 · Capture

Capture the winning pattern (+ any new root cause, + the experiment log for code-performance) to the brain.

## Output

- **Short path:** the root cause (one line) + the chosen fix + the measured check.
- **Full path:** root cause · the N ideas + scoring table · what you tested + the measured result · the chosen fix + its pre-mortem check + the honest re-score.
- **Code-performance path:** the profile excerpt naming the hot spot · the N ideas across ≥2 axes + scoring table · the experiment log with each statistical verdict · the kept change, its test-suite run, and `/honestscore`'s MIN with the performance dimension evidenced.

Never fix with the first idea. Never patch a symptom. Never claim a fix works — or is faster — without measuring it.