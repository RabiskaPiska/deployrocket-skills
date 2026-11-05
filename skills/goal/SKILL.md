---
name: goal
description: Goal-driven iteration engine — drive any declared goal to DONE by looping the full skill stack (ideas → specialist agents → subagents → verification → honestscore) until a compound stop condition fires. Use whenever the user declares a goal, target, or "keep going until X" — especially long-running autonomous pushes (48 hours to 10 days), loop-until-score runs (9.5/9.8), or any task needing unique ideas every iteration plus a 10–30 subagent crew. Routes each slice to a matching specialist agent when one exists (.opencode/agents or ~/.config/opencode/agents). Guarantees unique, non-repeated attempts via a fingerprinted idea ledger, survives context resets via file-based state, and never trusts self-reported done.
metadata:
  opencode/slash: "true"
---

# goal — Goal-Driven Iteration Engine

You are running the **Goal Skill**. A goal has been declared. Your job is not to attempt it once — it is to run a disciplined loop (unique idea → crew execution → external verification → honest score) until an EXHAUSTIVE, pre-frozen stop condition fires. The loop's memory lives in files, never in context.

## 0) HARD RULES

1. **The contract is frozen before run start.** Write `GOAL.md` + `goal-state.json` FIRST; the loop never edits its own exam mid-run. If the goal itself changes, that is a NEW goal: rewrite the contract, reset the stall counters, keep the ledger.
2. **Done is machine-checkable, never vibes.** The Definition of Done is ≤5 SHELL-CHECKABLE criteria (test passes, build exits 0, file exists, endpoint 200, score ≥ N printed by a script). If the goal cannot be expressed this way, split or sharpen it first — a natural-language "is it done?" check defaults to *not done* forever (AutoGPT lesson).
3. **Every iteration produces a UNIQUE attempt.** No idea enters execution without passing the Uniqueness Gate (§4) against the attempts ledger. Same idea in new words = repeat = rejected.
4. **Never trust self-reported success** — not the executor's, not a subagent's `status: success`. Verify real side effects: exit codes, files on disk, test output pasted. A claimed artifact with no artifact is a failed iteration.
5. **Every loop has BOTH an exit condition AND a hard cap**, and both are written in the contract before iteration 1.
6. **Stall ≠ pause.** Two iterations with no measured progress → forced replan into a different approach class. Three → escalate (§6). Never silently repeat a failed step.
7. **The orchestrator owns merge and verification.** Subagents never write the final deliverable and hand off via file pointers, never prose scrollback. Nested spawn is allowed ONLY when `experimental.subagent_depth` is ≥2 in the live config AND the child is a specialist whose `subagent` permission allows the named grandchildren — otherwise children must not spawn (default depth is 1; the check is §5a).
8. **A cap that trips while the goal is still actively progressing (no plateau declared) means the stop condition was mis-sized** — diagnose in the ledger before raising it. A declared budget cap firing at a plateau is a legitimate close (§6.2), not a defect.

## 1) Phase 0 — the Goal Contract (before iteration 1)

Write `GOAL.md` — a contract-shaped goal (davidondrej `/goal` pattern + Anthropic harnesses):

```
# GOAL: <one objective — ONE. This is not a backlog.>
## Read-first        — files/repos/docs the loop must load at boot
## Constraints       — stack, style, budget, out-of-scope list
## Validate          — THE one exact command (or ≤5) that proves done, with expected output
## Stop-when         — verifiable end state (ties to the DoD checks in goal-state.json)
## Budget            — iteration cap (default 200 — an honesty review point, NOT a cost saver; quota posture belongs to the operator), wall-clock cap (48h–10d), score target (the honestscore MIN the loop must reach, e.g. 9.5 or 9.8)
```

Write `goal-state.json` (JSON over Markdown — resists agent overwriting; Anthropic finding):

```json
{
  "goal": "...", "created": "<iso>", "iteration": 0,
  "dod": [{"check": "<shell-checkable>", "cmd": "<exact command>", "passes": false}],
  "score_target": 9.5, "verdict_allowlist": ["approved", "ready"],
  "stops": {"max_iterations": 200, "wall_clock": "<iso deadline>", "no_progress_replan": 2, "no_progress_escalate": 3},
  "status": "pursuing"
}
```

`passes` flags start `false`. The agent cannot quietly redefine done mid-run. Initialize `attempts.jsonl` (empty) and `progress.md` (boot notes, one section per iteration).

## 2) State layout (the loop's whole memory)

**All state lives in ONE run dir:** `.opencode/runs/goal-<slug>/` where `<slug>` = kebab-case first ~5 words of the goal, fixed at Phase 0. `<goal-slug>` everywhere below means this dir.

| File | Role | Written |
|---|---|---|
| `GOAL.md` | Frozen contract; re-injected every iteration (anti-drift) | Phase 0 only |
| `goal-state.json` | DoD flags, counters, stop config, status | every iteration end |
| `attempts.jsonl` | Idea ledger: `{n, fingerprint, hypothesis, test, outcome, delta, verdict}` | every iteration end |
| `progress.md` | Human-readable lab notes + decisions + blockers | every iteration |
| `HANDOFF.md` | Objective / Decisions / Blockers / Next moves / Files — written so a FRESH agent could resume | every iteration end |
| git | One commit per iteration. Pathspec is the run dir only: `git add -- <run-dir>` then commit. Never `git add -A` in a shared repo. Workspace not a repo → `git init` INSIDE the run dir (state-only repo); git unavailable → copy the state FILES (never the checkpoints dir itself) to a SIBLING dir `goal-<slug>-checkpoints/<n>/` instead | every iteration |

**Boot ritual (start of EVERY iteration, especially after a context reset/compaction):** read `GOAL.md` → `goal-state.json` → tail of `attempts.jsonl` → `git log --oneline -10` → run the Validate command(s). The agent is amnesiac; the filesystem is not.

## 3) The iteration loop

**P0 — CHECK DoD FIRST** (not last): run the Validate commands. If all DoD checks pass → go to P5 (score), then P6 (compound stop). Do not skip the score. Do not declare victory on green checks alone.

**P1 — RECON (first iteration, or when replanning):** fire `skill({id:"deep-thinker"})` — parallel multi-source research on the goal domain. If the goal means adopting external features/patterns, fire `skill({id:"github-feature-mine"})`. Produce a sourced evidence brief before ideating blind.

**P2 — IDEATE uniquely:** fire `skill({id:"ideas"})`. Diverge N ≥ 5 candidates across ≥2 axes (quick-fix / proper-fix / reframe / inversion / borrow-from-named-reference). Each candidate carries: fingerprint, one falsifiable hypothesis, the cheapest test that could refute it, kill-criterion. Pass every candidate through the Uniqueness Gate (§4). Score in one table (`idea | fixes-what | effort | risk | confidence | kill-criterion`), rank by evidence, shortlist top 1–2.

**P3 — EXECUTE via crew (§5):** route each disjoint slice to a specialist agent when one matches (§5a), else to `explore`/`general`. Spawn the crew in ONE message per wave. Delegation contracts only: objective + output format + tool guidance + explicit OUT + named deliverable + the JSON final-message contract. Fragments land under `.opencode/runs/<goal-slug>/out/`; orchestrator merges (contracts → implementations → consumers), verifying at every join.

**P4 — VERIFY externally:** in order (Anthropic grading hierarchy): (1) exec checks — run the Validate/build/test commands, paste real output; (2) de-correlated judge — a FRESH verification subagent that never saw the writing context, read-only, mandated to *refute* ("build the case this fails the spec"), different model family when available — scale to blast radius: trivial iterations may substitute a fresh-context refute-check performed by the orchestrator on the recorded artifact, substantive iterations always get a separate agent; (3) multi-judge consensus on fuzzy criteria — never the generator grading itself (LLM self-preference is measured, arXiv 2404.13076). Trust-but-verify every subagent envelope with side-effect checks.

**P5 — SCORE:** fire `skill({id:"honestscore"})` on the iteration's artifact. Record the MIN, the per-dimension vector, and the NOT-verified list into `progress.md`.

**P6 — COMPOUND STOP CHECK.** Stop = ALL of: every DoD check passes (exec-verified) · honestscore MIN ≥ `score_target` · verdict ∈ `verdict_allowlist` · all binary must-have gates MET · adversarial pass clean — zero findings, or all findings rebutted/closed within ≤2 rebuttal rounds; survivors of the cap become typed defect tickets reported openly at close. A high score with a "not ready" verdict does NOT stop the loop (ARIS lesson).

**P7 — RECORD + CLOSE iteration:** append the attempt to `attempts.jsonl` (fingerprint, hypothesis, outcome, score delta — every score raise must name the defect it closed); update DoD flags; rewrite `HANDOFF.md`; git commit; re-inject goal from `GOAL.md` (NEVER from a rolling summary — summary-of-summary drift is the top "did something I didn't ask for" cause); loop to P0.

Loop discipline: every iteration must clear at least one EXTERNAL pass/fail check; an iteration with no external check does not count. Score increases must name the defect closed. Loop `honestscore → ideas → fix → honestscore` inside each iteration until the bar or the 2-round re-verify cap — then next iteration.

## 4) Uniqueness Gate (every candidate, every iteration)

1. **Fingerprint** each idea: normalized approach signature = {approach class, target file/component, mechanism verb} — lowercase, whitespace-collapsed, then `sha256` the collapsed string (any deterministic one-liner, e.g. `python -c "import hashlib,sys;print(hashlib.sha256(sys.argv[1].encode()).hexdigest()[:16])" "<signature>"`).
2. **Ledger consult:** fingerprint matching a prior attempt whose `outcome` was `no-delta` or `rejected` → rejected on sight. A prior attempt that closed a named defect, or that died on a *different* named defect, is not a repeat — the fingerprint for a defect-chase is `{approach class, target, mechanism verb, defect-id}`. Same mechanism plus a new defect-id is a new attempt. Same fingerprint AND same defect-id as a no-delta row is a repeat.
3. **Semantic repeats count:** "same idea, new words" on the same defect is a repeat — compare mechanism, not phrasing. Hand the candidate + the last 10 ledger entries to the judge with one question: "same mechanism AND same defect as a no-delta row?" Only a yes on both rejects. A yes on mechanism with a different defect-id does not.
4. **Doom-loop guard:** hash normalized args+result of the iteration's key actions; 3 identical signatures = loop detected → inject a corrective reframe forcing a different approach class.
5. **Diversity floor:** each iteration must target a different approach class OR a named defect from the previous honestscore output (typed defect tickets drive the next attempt — this is how honestscore and ideas couple). Below diversity threshold → next attempt MUST use an unused approach class.
6. **Explore→exploit schedule:** while >50% of budget remains, force maximally distinct candidates; inside the last 50%, converge on the best-scoring line (BAVT bandit schedule).

## 5) Crew doctrine (10–30 subagents)

- **Size by genuinely-disjoint scope, then apply the owner floor:** a goal iteration runs a crew of 10–30 subagents ACROSS ITS WAVES (owner floor ≥10 on goal runs; trivial goals fill the 5 role seats — recon, cross-check, build, verify, adversarial; 20+ only for truly independent units). Never pad with duplicate scope — fill seats by ROLE. **Smoke-scale escape:** a trivial single-file/single-check goal may run one orchestrator-executed iteration with no crew — record the reason in the ledger; the 10–30 doctrine applies to real goal runs.
- **Concurrency per message ≤8** (one wave per message, agents in a wave run concurrently). 10–30 "at once" = one crew, multiple waves back-to-back within the same iteration — not 30 spawns in a single message.
- **Isolation:** parallel file-writing agents work in separate git worktrees or disjoint file territories; one file cluster = one owner. Artifact pointers (paths) over full-text handoffs; share full traces for decisions, not summaries (Cognition).
- **Effort-scaling rule in every brief:** "Report blockers in one line; do NOT widen scope. Do NOT spawn grandchildren unless §5a says nested spawn is live for THIS agent. The orchestrator owns merge."
- **Cap discipline:** ≤2 retries per agent; a silent finisher is chased via session resume; a killed agent's output is never merged. Budget ~10–15% of spawn cap for the verify reserve. Grandchildren, if nested spawn is live, count against the SAME 30-cap — they are not free.

## 5a) Specialist routing (custom agents)

OpenCode V2 loads specialists from `.opencode/agents/<id>.md` (project) and `~/.config/opencode/agents/<id>.md` (global). Frontmatter `mode` is `primary` | `subagent` | `all`. A missing `mode` on a new file defaults to **primary** — a goal worker must be `subagent` or `all`, or the spawn is rejected. `description` is what the model sees when picking who to launch; write it as a trigger, not a slogan.

**Before wave 1 of every iteration:**

1. List both agent dirs. For each file, read only the frontmatter (`description`, `mode`, `steps`, `model`).
2. Match a slice to an agent only when the slice's subject is inside that description. No match → `explore` (read) or `general` (write). Never force a specialist onto an adjacent domain.
3. Spawn with `subagent({ agent: "<id>", ... })`. The child inherits the session model unless its file sets `model`.
4. One file cluster still has exactly one owner. Two specialists that would edit the same files run in sequence, not parallel.

**Nested spawn (the "each expert spawns 30") — check, do not assume:**

Default `experimental.subagent_depth` is **1**. At 1, a specialist that calls `subagent` gets `Subagent depth limit reached`. Built-in `general` and `explore` cannot launch children even if depth is raised.

Nested fan-out is live only when ALL of these are true, verified this run (read the merged config, do not recall):

- `experimental.subagent_depth` ≥ 2 (3 if you want grandchildren to spawn too).
- The specialist's own `permissions` allow `action: subagent` for the grandchild ids it will use.
- The goal's remaining spawn budget can pay for those grandchildren. They count inside the 30, not beside it.

If any check fails, the specialist does the slice itself and reports the blocker in one line: `nested: off (depth=1)`. Do not tell the operator the tree fanned out.

**Shape that actually scales a C# (or any) build:** one orchestrator session running `/goal`. It spawns the matching specialists (AI / backend / frontend / algorithms / cyber) as **siblings** in waves of ≤8. Each specialist loads only its skills. The orchestrator merges. That is the 10/10 shape on stock OpenCode V2. Raising `subagent_depth` is optional parallelism, not a requirement, and it multiplies tokens (~15× per extra agent layer, Anthropic's measured figure) and duplicate-work failures. Do not raise it inside a goal run — the operator sets it in `opencode.jsonc` on purpose.

## 6) Stop conditions (exhaustive — written in the contract, checked in this order)

| # | Condition | Action |
|---|---|---|
| 1 | **ACHIEVED** — compound stop (P6) fully green | Close: final report + honest score + evidence; update brain; CONTINUE-HERE |
| 2 | **BUDGET** — wall-clock or iteration cap hit | Close honestly at current MIN score with open-findings list; never round up |
| 3 | **NO-PROGRESS** — 2 stalls → forced replan (new approach class + refreshed facts, Magentic-One); 3 stalls → escalate | Escalate with evidence: what was tried (ledger), what is blocked, what the operator must supply |
| 4 | **BLOCKED** — needs manual input only the operator can supply (credential, login, payment) | Name the ONE item precisely; do every other part first; never invent the input |
| 5 | **CONTRACT BROKEN** — DoD unsatisfiable as written | Diagnose in ledger; propose rewritten contract as a NEW goal (reset counters, keep ledger) |

Escalation is output with partial results — never a silent retry, never a truncate.

## 7) Long-run mode (48 hours – 10 days)

- **Relauncher (mandatory for multi-day runs — the loop must survive a dead process):** at Phase 0, write `relaunch.ps1` (Windows) or `relaunch.sh` into the run dir. It runs ONE iteration per invocation with a fresh `opencode run`, because the V2 CLI intro (https://opencode.ai/v2/docs/cli/, fetched 2026-10-22) documents `opencode run "<prompt>"` and does **not** document `--continue` / `--session`. Do not pass those flags. The prompt file must name the absolute run dir: "Boot ritual per goal skill §2. Read `<run-dir>/GOAL.md`, `goal-state.json`, the tail of `attempts.jsonl`, and `HANDOFF.md`. Run the Validate command. Do exactly one iteration. Stop."
 - **Status gate, first line of the script:** read `goal-state.json` `status`. If it is anything but `pursuing`, exit 0 and do not call `opencode`. Otherwise a scheduler keeps booting a finished goal until the wall-clock.
 - **Lock:** `run.lock` holds `{pid, startedUtc}`. On start, if the lock exists and that pid is still alive and younger than the wall-clock cap, exit 0 (another iteration is in flight). If the pid is dead, or `startedUtc` is older than the contract's per-iteration budget (default 6h when the contract names none), delete the lock and continue — a crash must not wedge the next fire. Write the lock after that check; delete it in a `finally` / `trap`. Two schedulers never overlap; a dead process costs one iteration, not the run.
 - Schedule with Task Scheduler / cron, or a `while` that also stops when status leaves `pursuing` or the wall-clock in the contract is past. `opencode run --help` on that box is the check — if a continue flag exists there, record it in the ledger, still keep the file boot as the path that works without it.
- **One iteration ≈ one session/context window.** On compaction or reset, the boot ritual (§2) + `HANDOFF.md` rebuilds full state; full-reset-with-handoff beats summarize-in-place for very long jobs.
- **Anti-drift:** goal re-injected from the original `GOAL.md` every iteration; never from rolling summaries.
- **Heartbeat:** any run >5 min gets a ~3-minute pulse verifying real side effects (new files, commit hash, growing counts); two flat beats = stall → recover per §6.3.
- **Checkpoint every iteration:** git commit + `goal-state.json` write = the resume point. Resume = a fresh `opencode run` whose prompt points at the run dir (see relauncher). The ledger + handoff make any fresh session equivalent. Do not depend on `--continue`.
- **Test ratchet:** once a DoD check passes, it may never be removed or weakened to keep later iterations green.

## 8) Skill combination map (fire by phase, not all at once)

| Phase | Skill(s) | Why |
|---|---|---|
| Contract / boot | `godmode` | activation protocol, every session |
| Recon / research | `deep-thinker`, `deep-research-plus` | multi-source evidence before ideating |
| External features | `github-feature-mine` | mine proven repo features when the goal adopts patterns |
| Ideation | `ideas` | N unique candidates, scored, kill-criteria |
| Execution | `subagents` | crew spawn/merge doctrine (§5) |
| Specialists | matching agent id from §5a | route the slice; do not invent an agent |
| Testing/repair | `deep-test-all` + language gate (`dotnet-quality-gate` / `py-quality-gate` / `cpp-quality-gate`) | when the artifact is code |
| Ship-gate | `web-security-hardening`, `verification-before-completion` | before any deploy claim (user-level skills — loadable via the skill tool even when not in the project skills dir) |
| Scoring | `honestscore` | the only score the loop trusts |
| Prompts | `promptgod`, `prompt` | when the goal's artifact is itself a prompt/skill |

## Output

- Per iteration: the unique attempt (fingerprint + hypothesis), crew log, pasted verification evidence, honestscore MIN.
- On stop: which stop condition fired, the final honest score (MIN/mean/vector), the NOT-verified list, and typed defect tickets for anything open.
- Final line of the turn, always: `goal — iteration: <n> | status: <pursuing|achieved|budget|no-progress|blocked|contract-broken> | dod: <passed/total> | honestscore: <MIN or in-progress> | unique: <fingerprint ok|rejected dups>`

`gates — godmode:activation: <fired|skipped: why> | /honestscore: <fired N.N|skipped: why> | /ideas: <fired|skipped: why> | brain: <file(s) written|skipped: why>`

Read when the phase needs it, not up front:
- `references/research-catalog.md` — the 20 mined loop skills and the 2026 sources.
- `references/specialist-agents.md` — where agent files live, why omitted `mode` means primary, the depth-2 switch, the VPS copy path.
