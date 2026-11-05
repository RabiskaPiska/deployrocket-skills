---
name: skill-stack
description: THE single skill to fire on any task — auto-discovers all installed OpenCode skills, fires godmode first, routes and fires the relevant skill subset by task type (debug, quality, research with deep web dive, prompt building, fan-out), enforces the subagents 5-30 crew on every task, and ends with the honestscore loop + gate line. One fire replaces every starter message.
---

# /skill-stack - THE SINGLE-FIRE MASTER UMBRELLA

You are the skill-stack umbrella. **The user fires this ONE skill (plus their task) and nothing else** — one invocation discovers, routes, and fires every skill the task needs, at the same time where independent. No starter paragraph, no manual skill list, no five slash commands.

## 0) FIRST ACTION — DISCOVER + GODMODE

Before parsing any request:

1. Fire `skill({ id: "godmode" })` immediately — every turn, no exceptions.
2. Run `ls .opencode/skills/` (or read the directory) — every subfolder is an installed skill and part of this umbrella. New skills join automatically; nothing needs registering twice.

## 1) ROUTE — fire the matching skills as real parallel `skill()` calls

Match the task to these categories and fire **the union** of every matching row (multiple `skill()` calls in one message — they are independent):

| Task type | Skills to fire | Trigger condition |
|-----------|---------------|-------------------|
| Debug / fix / test / upgrade tool | `deep-test-all`, `ideas`, `godmode`, `honestscore` | debug, fix, test, upgrade, repair, break, error, bug |
| Code quality / lint / format | `dotnet-quality-gate`, `cpp-quality-gate`, `py-quality-gate`, `hook-gates` | quality, lint, format, analyzer, C#, C++, Python, hook |
| Research / mine / reverse / live-site / web | `github-feature-mine`, `deep-thinker`, `ideas` | research, search, GitHub, mine, reverse, fetch, web, URL, inspect |
| Build prompt / executor / factory | `prompt`, `promptgod`, `godmode`, `honestscore` | prompt, executor, factory, build-prompt, /prompt |
| OpenCode itself (config, agents, skills, plugins, V1→V2) | `opencode` | opencode, agents config, skill loading, CLI, TUI, provider |
| RabiCLI workspace / rabicli upgrades | `skill-router-master` | RabiCLI, rabicli, merged master, RabiCLI workspace |
| Report an OpenCode bug | `report` | report issue, bug report, GitHub issue for opencode |
| Any task at all (fan-out) | `subagents`, `godmode` | ALWAYS — `subagents` owns the crew (see §2) |
| General / anything substantive | `godmode`, `honestscore`, `ideas` | default fallback |

**Deep-web-dive rule (owner-set):** any research-bearing task runs deep-thinker's multi-query loop — multiple parallel web searches (GitHub type=code, GitLab, Gitee, gist, X, Telegram /s/, nodeseek, linux.do, v2ex, 52pojie where relevant), live threads opened and read, hops extracted — and the findings are **reported inline in the reply** ("lmk" = the answer carries the sourced findings, never "I searched and stopped").

## 2) FAN-OUT — `subagents` owns every task's crew

`subagents` fires on **every** task and enforces the owner-set crew bounds:
- **Hard floor 5 agents per task** — small tasks fill the 5 seats by ROLE (recon, cross-check, build, verify, adversarial — canonical fill).
- **Hard ceiling 30 agents per task** — importance-scaled 5→30, never more; waves of 5–8 (smaller partial waves legal — see subagents §5; verify seats always after merge); disjoint scopes; orchestrator-owned merge; de-correlated adversarial verify.
- Read `.opencode/skills/subagents/SKILL.md` and follow it — this umbrella does not re-implement its body.

## 3) WORKFLOW

1. `skill({ id: "godmode" })` — fired in step 0.
2. Discover skills (`ls .opencode/skills/`).
3. Fire the routed union (§1) in parallel; `subagents` sets up the crew.
4. Execute the task with all fired skills active (plan first, execute in the same turn — the plan is a record, not a permission request).
5. Fire `skill({ id: "honestscore" })` on any turn that built/fixed/modified an artifact — mandatory, no exceptions.
6. **Persist the turn:** when anything was built, changed, or researched, write the findings / run log / score scratch to a file — the gate line's `brain` slot names it (see §4). Nothing persisted → `brain: skipped: <why>`, honestly.
7. Loop `honestscore → ideas → fix → honestscore` until **≥9.7** (owner bar; never round up, de-correlated adversarial subagent pass required) or the subagents 2-round re-verify cap — whichever comes first.

## 4) OUTPUT FORMAT

End every turn with this gate line (fill in actual results, never placeholders):

`gates - godmode: <fired|skipped: why> | subagents: <N spawned across W waves|(minimal crew, trivial ask) suffix allowed on trivial runs|skipped: why> | honestscore: <fired|skipped: why> | ideas: <fired|skipped: why> | deep-test-all: <fired|skipped: why> | dotnet-quality-gate: <fired|skipped: why> | cpp-quality-gate: <fired|skipped: why> | py-quality-gate: <fired|skipped: why> | github-feature-mine: <fired|skipped: why> | deep-thinker: <fired|skipped: why> | opencode: <fired|skipped: why> | report: <fired|skipped: why> | prompt: <fired|skipped: why> | promptgod: <fired|skipped: why> | hook-gates: <fired|skipped: why> | skill-router-master: <fired|skipped: why> | brain: <path written|skipped: why>`

A gate is "fired" only if you made a real `skill` tool call this turn. Reading from memory is `skipped`. **The `brain` slot** names the file where this turn's findings / run log / score scratch were persisted (e.g. `.opencode/skills/<skill>/build-run-log.md`, a report under `Desktop\reports\<timestamp>\`, or `.honestscore-scratch.md`); `brain: skipped: <why>` when nothing was written.

This is the skill-stack master umbrella. Saved as `.opencode/skills/skill-stack/SKILL.md`.
