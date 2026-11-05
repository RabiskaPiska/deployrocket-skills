---
name: proven-working
description: High-stakes verification protocol. Invoke for changes where a mistake is expensive or hard to reverse (migrations, auth/security changes, release-blocking fixes, anything the user flags as "go deep" / "proven working" / "100% mode"). Runs a multi-check gate before delivery and uses a strict, evidence-first delivery format.
---

# Proven-Working Protocol

Use this when correctness matters more than speed. The goal: every claim you ship is backed by evidence captured **in this session**, not by reasoning alone. Scale the rigor to the task — a two-line fix needs the spirit of this, not all ten pages of it.

## Step 1 — Decompose (think first)

List what the change actually touches:
1. **Primary** — the explicit feature/bug + every file, function, and dependency implicated.
2. **Secondary** — tests, logging, error handling, migrations, CI, rollback that the change implies.
3. **Risks** — breaking changes, perf regressions, security holes, data-integrity hazards, dependency conflicts.
4. **Edge cases** — empty/null inputs, Unicode, concurrency, partial failure, resource exhaustion.

Keep this proportional. A small fix is a few bullets; a migration is a real list.

## Step 2 — Scan the workspace before researching

Use Glob/Grep/Read (and the Explore subagent for recon) to learn the project's actual versions, patterns, and conventions. Read dependency files in full (`pyproject.toml`/`uv.lock`, `*.csproj`/`global.json`, `go.mod`/`go.sum`, `package.json`/lockfile) — note exact versions, flag CVE-affected packages. Match the existing style; don't impose a new one.

## Step 3 — Research only what you can't verify locally

When docs/behavior aren't knowable from the code, research it. **No fixed quota** — stop when independent sources converge (3+ agreeing, or one authoritative primary source) and further searches stop adding signal. Prefer official 2025–2026 docs for the *exact version in use*; WebFetch full pages rather than trusting snippets. For a large multi-topic investigation, invoke `/deep-research-plus`.

## Step 4 — The verification gate

Run the checks that apply. A change is ready when the load-bearing ones pass with evidence:

| # | Check | Evidence it produces |
|---|-------|----------------------|
| 1 | **Code trace** | You read the actual symbols/signatures/call sites — quote them. |
| 2 | **Terminal test** | Build/test/lint run, exit code + output captured. (If you can't run it, give the exact command.) |
| 3 | **Official docs** | The exact version's docs, by URL. |
| 4 | **Independent corroboration** | 2–3 agreeing recent sources, or one primary source. |
| 5 | **Regression** | Callers/dependents traced; nothing currently passing now fails. |
| 6 | **Security** | New deps checked (NVD/advisory); input paths checked for injection; auth surface checked. |
| 7 | **Edge cases** | ≥2 concrete failure scenarios reasoned through or tested. |

**Critical four — never skip:** code trace (1), regression (5), security where relevant (6), and terminal test (2) for anything executable. If a critical check can't pass, say so honestly — don't ship and don't pretend.

Keep the review **scoped to correctness and the stated requirements**. A reviewer hunting for "any problem" invents findings and pushes over-engineering — don't.

## Step 5 — Deliver (evidence-first)

```
## Summary
One paragraph: what the change is, why it's proven, key findings.

## Verification evidence (per component)
✓ Code trace — what you read
✓ Terminal test — command + exit code + output (or the exact command to run)
✓ Docs — version + URL
✓ Regression — callers traced
✓ Security — what you checked
Result: which checks passed.

## Changes
Full file paths. Complete edits — no "// rest unchanged", no placeholder TODOs, full imports. Production-ready.

## How to verify
The exact commands that prove it works.

## Risks & rollback
Breaking risks (severity), migration steps, rollback plan. Numbered next steps or precise questions if anything's genuinely ambiguous.
```

## Step 6 — Post-delivery sweep

Re-read your own diff for syntax/imports/consistency. Mentally run the changed path end-to-end. List the symbols you modified and confirm every caller still works. Flag anything left uncertain — don't bury it.
