---
name: context-budget-manager
description: Per-turn context-fill accounting and cumulative token/cost tracking for every session - estimates context consumption each turn, logs a cumulative ledger, warns at 40-50% fill, and auto-invokes context-handoff/compaction at the threshold math (window minus reserve), closing the GodMode "no token self-awareness" gap. Cites real measured runaways ($59/3h) and 2026 harness doctrine (LangChain gateway budgets, Braintrust kill switches, Claude SDK maxBudgetUsd, MS Agent Framework compaction). Triggers on: any substantive task, long autonomous runs, "how much context is left", "am I going to compact", token/cost budget questions, any always-fire turn.
alwaysFire: true
---

# context-budget-manager - Per-Turn Context Accounting

## 0. PURPOSE

The fleet's own memory states it plainly: the model has "no native token-budget
self-awareness" (GodMode protocol, CONTEXT HYGIENE section). Models cannot see their own
window fill. Measured consequences already on record:

- "$59/3h silently" runaway (skill-circuit-breaker, measured risk).
- Sessions compounding fixes in polluted threads until a hard reset was forced.
- Context rot measured at 20-30K tokens of degradation before any compaction trigger
  fires (skill-context-handoff: quality degrades at 20-30K per NoLiMa/RULER research).
- Handoffs written too late: "compact at window-50%, not 90%" is the recorded rule, and
  it is violated whenever nothing tracks the fill.

This skill makes context consumption a NUMBER every turn, with thresholds, a ledger, and
a pre-armed compaction path. It is the always-fire declaration layer (RGAP2 gap #1) plus
the state-snapshot-naming primitive (gap #2): at task start you name the state file;
on stall, the monitor restores from it.

## 1. WHEN TO FIRE (always-fire)

Fires on EVERY substantive turn, unconditionally:

- Every multi-step build/fix/research task (the GodMode substantive tier).
- Every turn after the first in any session longer than 10 turns.
- Every run with subagents (their returned summaries are context too).
- Explicitly on: "how full is the context", "budget check", "are we over budget",
  "compact now", "hand off", and before any /clear, /new, or compaction.

Trivial one-off answers may skip the ledger but not the awareness: if the session is
already past the warn threshold, even a trivial answer gets a one-line budget note.

### Always-fire justification

The failure this prevents (silent context exhaustion mid-task) recurs across every task
type and always breaks silently -- the defining always-fire test from RGAP1 section 4.
Cost-side corroboration is web-verified in runlog-RGAP2: Claude Agent SDK ships
first-class total_cost_usd + maxBudgetUsd with resume carryover; LangChain routed all
coding-agent calls through a gateway with per-workspace budgets after unbounded spend
went unnoticed for a week; Braintrust 2026 prescribes hard caps + kill switches. The
harnesses made budget first-class; this skill is the fleet's equivalent at the skill
layer, paired with hooks (skill-circuit-breaker) for the mechanical enforcement.

## 2. THE ACCOUNTING MODEL

### 2.1 What fills the window (in rough cost order)

| Source | Typical size | Notes |
|---|---|---|
| System prompt + harness instructions | 3-8K tokens | Fixed per session; re-injected after compaction |
| Always-loaded rules/AGENTS.md | 2-6K | The 200-line doctrine exists to cap this |
| MCP/tool catalogs | 2-10K | Per enabled server; the catalog header alone is measurable |
| Memory index (MEMORY.md) | 1-4K | Progressive disclosure keeps detail files out until read |
| File reads (large) | 1-50K each | The #1 uncontrolled contributor; delegate wide reads to subagents |
| Tool outputs (untruncated) | 0.5-20K each | Search results, HTTP bodies, log dumps |
| Subagent returns | 1-2K each (if distilled) | 10K+ if the mandate failed to demand a summary |
| Skill bodies fired | 0.5-5K each | 21 always-fire skills make this structural |
| The conversation itself | grows every turn | Compaction is the only release valve |

### 2.2 The ledger (where the numbers live)

Path: `<workdir>/context-budget-ledger.md` (create on first fire; append-only).

```markdown
# CONTEXT-BUDGET LEDGER - <session-slug>
window=<model window tokens> reserve=<reserve> warn_at=<fill%> compact_at=<fill%>
state_file=<path to the task's state/NOTES file>

| turn | est_cum_tokens | fill% | biggest_add (what, size) | action taken |
|------|----------------|-------|--------------------------|--------------|
| 1    | 12000          | 6%    | system+rules 9K          | -            |
| 2    | 31000          | 16%   | file read 14K            | delegated next read |
```

Rules:
- est_cum_tokens is an ESTIMATE and is labeled as such; it is recomputable from the
  ledger rows (sum of additions), never from a feeling.
- Every row names its biggest addition. A row that cannot name one was not measured.
- The ledger survives compaction: it lives on disk, and the post-compaction turn
  re-reads it and continues the table (reattach-after-compaction doctrine).

### 2.3 Estimation without a token meter (the honest method)

No model self-reports tokens reliably. Estimate by structure, and mark it estimated:

- Char/4 for English text is the coarse default (a 12,000-char file read ~= 3K tokens).
- JSON tool output: chars/3 (denser tokens).
- Code: chars/3.5.
- When the harness exposes real usage (e.g. context get-usage credits, session
  token fields in opencode storage, total_cost_usd in SDK runs), prefer the REAL
  number and record "measured" vs "estimated" in the ledger. A measured number
  replaces the estimate; do not run both.

## 3. THRESHOLDS (the math, stated once)

```
window        = the model's context window (tokens)
hard_reserve  = reserve for output + tool calls + the safety margin the harness
                already clamps fleet-wide (autocompact clamp = window - 33000)
soft_reserve  = headroom wanted for a clean handoff (the handoff document itself,
                plus the tail of live instructions that must be reattached)

fill%         = est_cum_tokens / window * 100
warn_line     = 40-50% fill        -> warn + start delegating wide reads
handoff_line  = window - hard_reserve - soft_reserve, expressed as fill%
                (the skill-context-handoff "compact at 50% of window" rule,
                 made concrete: do NOT ride to the auto-compact trigger)
stop_line     = 80% fill           -> stop starting new work; hand off NOW
```

### 3.1 Worked threshold table (common windows)

| window | hard_reserve (clamp) | soft_reserve | handoff_line tokens | handoff_line fill |
|---|---|---|---|---|
| 128K   | 33K                  | 16K          | 79K                 | ~62%              |
| 200K   | 33K                  | 20K          | 147K                | ~73%              |
| 200K (conservative, rot-aware) | 33K | 60K | 107K | ~53% |
| 750K   | 33K                  | 40K          | 677K                | ~90%              |
| 900K (fleet clamp ceiling) | 33K | 45K | 822K | ~91% |
| 967K (clamped max) | 33K | 48K | 886K | ~92% |

Context-rot adjustment: quality degrades well before the window is full (the 20-30K
NoLiMa/RULER finding in skill-context-handoff). For high-stakes work, use the
conservative row: hand off around 50-60% of the window regardless of the math, because
the last 30% of the window is where corrections start failing and the two-failed-
corrections rule fires. The math gives the CEILING; rot gives the PRACTICE.

## 4. PER-TURN PROCEDURE

Run at the END of every substantive turn (and at START when resuming):

**4.1 Count the turn's additions.** Walk the turn: file reads (sum chars/4), tool
outputs (chars/3 for JSON), subagent returns, skill bodies loaded. One line per source.

**4.2 Update the ledger.** Append the row: turn number, new cumulative estimate, fill%,
biggest addition. If fill crossed a threshold since last row, the row's action column
records what you did about it (that is the audit trail honestscore reads).

**4.3 Compare against thresholds.**

- fill < 40%: proceed normally. One ledger row. Done.
- 40% <= fill < handoff_line: WARN. Emit the budget line (4.6). Delegate any file read
  wider than ~2K expected tokens to a subagent with a 1-2K summary mandate. Stop
  quoting whole files into the main thread.
- handoff_line <= fill < stop_line: invoke skill-context-handoff NOW. Write the
  11-section handoff file BEFORE starting any new sub-task. Compaction may destroy the
  active instructions; the handoff is the reattachment source.
- fill >= stop_line: no new mandates. Finish or checkpoint the current unit, emit the
  handoff, and recommend /clear to the operator (or execute the compaction ritual
  yourself if the harness supports it: handoff -> /clear -> resume from file).

**4.4 Name the state file (gap #2 from the research).** Every task start writes one
line to the ledger: `state_file=<path>`. On stall, session-monitor-auto restores from
that file. This is the snapshot/restore primitive top harnesses ship as a distinct
recovery primitive (opencode-reconstituter / opencode-recover-project / omp /handoff);
here it is one naming line plus the file discipline, not a new subsystem.

**4.5 Cost side-accounting (when the session spends money).** If the run uses paid
model calls beyond trivial chat, append the budget declaration (from
skill-autonomous-run-budget, YAML) at task start and the breaker threshold at 80%.
One line in the ledger per beat: est spend, breaker state. The $59/3h lesson is that
"it is running" without a cost number is an unbounded run.

## 5. COMPACTION / HANDOFF RITUAL (the threshold fired -- now what)

1. Write the handoff file FIRST (skill-context-handoff format: 11 sections, dated
   immutable file + LATEST.md pointer + append-only PROGRESS.md).
2. Include: goal, DONE (with evidence paths), UNFINISHED, exact next steps, file
   locations, current ledger tail (so the next session inherits the fill history),
   and the state_file path.
3. Protect the recent tail: the last 2-3 turns of working context stay verbatim in the
   handoff's "current work" section (the omp /handoff pattern: compact in place with a
   protected recent tail).
4. Reattach after compaction: the first post-compaction turn re-reads (a) the handoff,
   (b) the ledger, (c) any always-on instruction the harness dropped. MS Agent
   Framework doctrine, verified 2026-10/11: "reattach active instructions after
   compaction" is the step everyone skips.
5. Verify the reset: new fill estimate should drop by the compaction savings; log the
   before/after in the ledger. If fill did NOT drop, compaction did not happen -- do
   not continue as if it did (verify-by-side-effect, not by claiming it).

## 6. VERIFICATION COMMANDS

The ledger is recomputable. These commands prove it:

```powershell
# Windows (this PC): row count and last fill
Get-Content .\context-budget-ledger.md | Measure-Object -Line
Get-Content .\context-budget-ledger.md -Tail 5

# Recompute the cumulative estimate from the rows (method stated, recomputable):
python -c "import re,sys; rows=[l for l in open('context-budget-ledger.md') if re.match(r'\| \d+',l)]; print('rows',len(rows)); print('last_est', rows[-1].split('|')[2].strip())"
```

```bash
# Linux/VPS: same checks
awk -F'|' '/^\| [0-9]+/ {n=$3} END {print "last_cum=" n}' context-budget-ledger.md
grep -c '^| ' context-budget-ledger.md   # row count = turn coverage
```

Pass criteria:
- Every substantive turn has a row (row count within 1 of substantive turn count).
- Last row's fill% matches est_cum/window arithmetic (recompute it; rule 2b-3:
  every number is recomputable from files that exist).
- Threshold crossings have an action recorded IN the crossing row, not in a later one.

Fail criteria: a session past handoff_line with no handoff file = FAIL, regardless of
what else was delivered. This mirrors "could not measure is NEVER a pass".

## 7. KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---|---|---|
| "It felt like plenty of room" then hard stop mid-write | No ledger; estimation skipped | Ledger is mandatory on substantive turns; retro-fill from transcript if needed |
| Ledger exists but rows stopped mid-task | Ledger updated manually and abandoned | Write the row FIRST, then do the work; the row is cheap, the rot is expensive |
| Handoff written but next session cannot resume | Handoff lacked file paths or exact next steps | The 11-section format exists for this; section names are mandatory |
| Two failed corrections on one problem | Context rot past the practical line | Stop patching; handoff -> /clear -> sharper spec (GodMode RECOVERY rule) |
| Compaction fired and skill instructions vanished | Reattach step skipped | Step 5.4 of the ritual; re-read the always-fire list post-compaction |
| Spend grew while "everything was fine" | No cost line in beats | Add est-spend to every beat in long runs (circuit-breaker pairing) |
| Subagent returns were 10K+ dumps | Mandate did not cap the return size | Mandates say "return 1-2K distilled"; enforce in subagents skill usage |
| fill% claims 25% but compaction fired | Estimate diverged from reality | The harness clamp is ground truth; recalibrate the estimator (chars/4 -> measured) |

## 8. INTEGRATION (dedup map -- who owns what)

| Concern | Owner | This skill's role |
|---|---|---|
| Compaction mechanics | harness-level auto-compact (fleet clamp window-33k) | This skill decides BEFORE the clamp: proactive, not reactive |
| Handoff file format | skill-context-handoff / skill-handoffs-upgrade | This skill TRIGGERS them at threshold; does not reimplement |
| Cost breakers | skill-circuit-breaker, skill-autonomous-run-budget | This skill carries the per-turn declaration line; breakers stay in-process |
| Stall/watchdog | session-monitor-auto, fleet-monitor | They consume the state_file named in 4.4 for restore |
| Subagent context hygiene | context-curator | This skill sets the 1-2K return cap that curator fills toward |
| Filling subagent context | context-curator | Complementary: curation keeps payloads small; budget tracks the totals |

No overlap conflict: every listed skill keeps its job; this skill is the accounting
layer that knows the numbers and fires the right one at the threshold.

## 9. SELF-TEST (red-green; a guard must fail on its own defect)

```
RED: open any long session transcript from before this skill; count turns since the
    last handoff; compute rough fill at the turn where quality visibly degraded
    (two failed corrections). If fill at that turn > handoff_line, the pre-skill
    harness had NO gate -- that is the defect reproduced.
GREEN: with the ledger active, the crossing row records the warn/handoff action, and
    a handoff file exists at or before handoff_line in post-skill sessions.
ESCAPE-HATCH PASS (10 min): (1) trivial-turn bypass -- confirm the one-line budget
    note still fires above warn even on trivial turns; (2) ledger tamper -- delete a
    middle row, confirm the recompute command flags the gap (row count vs turn
    count mismatch); (3) compaction no-op -- simulate a claimed compaction with no
    fill drop and confirm the ledger marks it FAILED rather than continuing.
```

## 10. GATES LINE USAGE

Ends every fired turn with the canonical 14-field line (gate-format-spec v14.2;
producer rule: never reorder, drop, or add fields; values never contain "|";
not-fired = "skipped: <why>"; n/a needs a one-line why):

```
gates -- godmode:activation: <fired|skipped: why> | promptgod: <fired|skipped: why> | subagents: <N seats|skipped: why> | honestscore: <N.N|skipped: why> | ideas: <fired|skipped: why> | deep-thinker: <fired|skipped: why> | dim-correctness: <N.N|skipped: why> | dim-completeness: <N.N|skipped: why> | dim-robustness: <N.N|skipped: why> | dim-security: <n/a: no external input this turn> | dim-performance: <N.N|skipped: why> | dim-concurrency: <n/a: single-session accounting> | dim-docs: <N.N|skipped: why> | brain: <files written | skipped: why>
```

This skill's own field contributions: the ledger path and last fill% belong in the
turn's STATUS/table output; a turn that crosses a threshold and does not say so in the
gates line's honesty fields fails the gate.

## 11. WORKED EXAMPLE

```
Turn 12, GLM session, window 967K (fleet clamp): ledger shows est_cum=412K, fill=43%.
Warn line crossed. Action recorded in row 12: "wide reads delegated; biggest add:
subagent return 2.1K".
Turn 19: est_cum=655K, fill=68% > conservative handoff (53%). Action: handoff file
written (docs/handoffs/2026-11-12-context-handoff.md), /clear executed, fresh session
resumes from handoff + ledger. Row 20 records fill reset to 9% post-compaction with
the before/after numbers pasted.
What the old behavior looked like: the same session rides to the auto-compact trigger,
loses the active instructions mid-shard, and the shard re-runs blind (measured in the
campaign as the stale-/tmp and clobbered-results class).
```

## 12. FAQ

Q: Is this not redundant with harness auto-compact?
A: No. Auto-compact is reactive and fires near the clamp; quality is already degraded
(user's own compaction threshold math: window-50%, not 90%). This skill is the
proactive twin, and it produces the ledger that makes the degradation visible.

Q: Why not a hook instead?
A: Partly hooks, by design. The mechanical parts (deny over-budget tool spam, budget
declaration) belong in hooks per Anthropic doctrine; this skill owns the parts needing
judgment: what to delegate, when the practical handoff line arrives despite a big
window, and what the handoff must contain. RGAP2's structural note applies: the fix
for a heavy chain is not "more always-fire prose" but this skill is deliberately thin
-- a ledger row and a threshold check, with the heavy lifting in existing skills.

Q: What if the model genuinely cannot estimate tokens?
A: Then it counts characters (rule 2b-3: recomputable with the method stated) and
labels the row ESTIMATE. The alternative -- no numbers at all -- is the measured
failure this skill exists to end.

## 13. VERSION / CHANGELOG

```
context-budget-manager v1.0 (2026-11-12)
- Authored from runlog-RGAP2 (gaps 1 and 2: budget enforcement and state-snapshot
  naming missing from the always-fire chain; compaction doctrine from MS Agent
  Framework + DenisSergeevitch compaction notes, verified 2026-11-12).
- Thresholds reconciled with fleet clamp (window-33000) and skill-context-handoff
  (compact at 50% window, not 90%).
- Cost side cites the fleet's own measured runaway ($59/3h) plus SDK/LangChain/
  Braintrust 2026 sources already verified in runlog-RGAP2.
```

## 14. OPERATOR QUICK CARD (the whole skill in 12 lines)

```
1. Task start: create/refresh ledger; write state_file line.
2. Every turn end: add one ledger row (est_cum, fill%, biggest add, action).
3. fill < 40%: nothing else needed.
4. 40-50%: warn; delegate wide reads to subagents (1-2K summaries only).
5. >= handoff_line: write the 11-section handoff BEFORE any new work.
6. >= 80%: no new mandates; checkpoint and hand off.
7. Paid runs: one cost line per beat; breaker armed at 80% of declared budget.
8. After any compaction: re-read handoff + ledger; log the before/after fill.
9. Claims about context ("plenty of room", "nearly full") REQUIRE a ledger row number.
10. Two flat beats on the ledger = stalled session; monitor recovers from state_file.
```

## 15. SUBAGENT ACCOUNTING (the hidden multiplier)

Subagents are the best context trade in the fleet AND the easiest way to blow the
window when misused. Rules:

- Every subagent mandate states a return cap: "final message = distilled summary,
  1-2K tokens max; raw reads stay in the subagent."
- The orchestrator counts a subagent's return as a single ledger entry
  (`subagent return, <id>, ~<K> tokens`). If a return arrives over cap, the entry
  says OVER_CAP and the next mandate tightens the cap -- the ledger is also a
  quality check on delegation.
- Fan-out math (from memory: use-multiple-agents): 8 subagents returning 1-2K each
  is ~16K of distilled context; the same work read into the main thread is often
  80-200K. The ledger row makes the trade visible instead of assumed.
- Never paste a subagent's full tool output into the parent. If a detail matters,
  the subagent writes it to a file and the parent cites the path.

## 16. MULTI-SESSION / FLEET CONTEXT

Long goals span sessions and boxes. The ledger is per-session; the fleet-level view:

- Each goal dir gets ONE cumulative line per session in its tasklist/PROGRESS file:
  `ctx: <session> last_fill=<N>% peak=<M>% handoffs=<k>`.
- A fresh session on any box reads the handoff + ledger tail and continues the
  numbering (CONTINUE-HERE protocol). The fill history travels with the handoff so
  the next seat inherits the burn rate, not just the task state.
- Watchdogs (session-monitor-auto) may read the ledger as a stall signal: a ledger
  that stopped growing while the session is active is the same "frozen job looks
  like a working one" failure as any other job -- two flat beats, then recover.

## 17. THRESHOLD RESPONSE PLAYBOOK (what exactly happens at each line)

| Crossing | Immediate action | Forbidden action |
|---|---|---|
| 40% | Warn line in the turn output; ledger row action="WARN"; switch reads to delegated mode | Quoting file bodies into the parent |
| 50% (conservative handoff line for high-stakes work) | Draft the handoff skeleton; finish the current unit; write handoff | Starting a new subtask "real quick" |
| handoff_line (window math) | Handoff file + /clear or compaction ritual, no exceptions | "one more step first" |
| 80% (stop_line) | Checkpoint state file; emit handoff; stop | Any new subagent spawn |
| Post-compact turn 1 | Re-read handoff + ledger; reattach instructions; log before/after fill | Trusting that memory survived (it did not -- that is what compaction is) |

The forbidden column is the point: every recorded incident in this fleet's notes
happened while someone did the forbidden thing "just this once".

## 18. LEDGER EXAMPLE (full, for a real run)

```markdown
# CONTEXT-BUDGET LEDGER - a11y-out adoption-skills run
window=967000 hard_reserve=33000 soft_reserve=48000 handoff_line=884000 (91%)
conservative_handoff=50-55% | state_file=./NOTES-s4.md

| turn | est_cum_tokens | fill% | biggest_add (what, size)        | action taken |
|------|----------------|-------|---------------------------------|--------------|
| 1    | 14000          | 1%    | harness instructions 8K         | none needed  |
| 2    | 26000          | 3%    | source runlogs read 12K         | none         |
| 3    | 31000          | 3%    | grep results 2K                 | none needed  |
| 4    | 33500          | 3%    | write sync-prune draft 12K      | noted        |
| 5    | 39000          | 3%    | skill expansion edit 8K         | none         |
```

Reading the example honestly: on today's huge windows the fill percentages stay
small for a long time. That is exactly why the ledger records ABSOLUTE adds too --
the rot risk is conversation LENGTH and instruction mass, not raw percentage, so the
turn-count and biggest-add columns carry as much signal as fill%. Both numbers go in
every row; neither is optional.

## 19. INTERFACE CONTRACT (for other skills/hooks)

Inputs consumed by this skill:
- Model window size (from harness config or the session's model card).
- Optional live usage sources when present: context get-usage (credits), session
  token fields, SDK total_cost_usd. When present, they outrank estimates.

Outputs produced by this skill (consumed by):
- `context-budget-ledger.md` rows (honestscore evidence; goal-loop progress checks).
- `state_file=<path>` line (session-monitor-auto restore path; CONTINUE-HERE writer).
- Threshold signals: WARN / HANDOFF / STOP (goal skill's loop controller, dag
  dispatcher pause, session-monitor-auto corrections).

Failure of the interface (no ledger where one should be) is itself a finding: log it
to the brain runbook, same as any other missing guard.

## 20. COMPACT vs HANDOFF vs /clear -- DECISION MATRIX

| Situation | Context coherent? | Someone must resume? | Action |
|---|---|---|---|
| Past 40-50% warn, work still clean | yes | not yet | Keep going; delegate wide reads; ledger only |
| Past handoff_line, mid-unit | yes | yes, soon | Finish current unit, then handoff + /clear |
| Past handoff_line, between units | yes | yes | Handoff file NOW (cheapest boundary there is) |
| Two failed corrections on one problem | no (rot) | yes | Stop patching: handoff + sharper spec + clean session (GodMode RECOVERY rule) |
| Unattended/overnight run ending a phase | n/a | yes, always | Handoff + CONTINUE-HERE refresh BEFORE stopping |
| Cost over budget (paid run) | n/a | n/a | Circuit-breaker trip (skill-circuit-breaker); this skill logs the spend line that tripped it |

Post-action verification, every row: the first turn after the action re-reads handoff
+ ledger and logs before/after fill. A compaction whose effect was never measured is
the rule-2b-5 trap: "0 pass / 1 defect / 2 unmeasured".

## 21. GLOSSARY

- window: the model's maximum context in tokens.
- clamp: the fleet's autocompact trigger (window minus 33000; ceiling 967K).
- fill%: est_cum_tokens / window, labeled ESTIMATE unless harness-measured.
- handoff_line: window minus hard_reserve minus soft_reserve (the must-handoff count).
- state_file: the one file a monitor restores from after a stall.
- distilled return: subagent final message capped at 1-2K tokens; raw material stays
  in files, never in the parent window.
- flat beat: a heartbeat period with no new verified side-effect; two in a row is a
  stall and triggers recovery.

## 22. VERSION HISTORY

```
v1.0  2026-11-12  initial (see section 13)
```

## 23. FLEET LEDGER PATHS (per host)

| Host | Ledger path | Notes |
|---|---|---|
| Windows PC | `<workdir>\context-budget-ledger.md` | workdir = the session's task dir; scratch scripts to the session scratchpad, never loose |
| VPS1 | `<goal-dir>/context-budget-ledger.md` | travels with the goal dir; survives session moves |
| VPS2 | `<goal-dir>/context-budget-ledger.md` | same; the dual-root gotcha does NOT apply (ledger is session state, not config) |
| Background crews | one ledger per seat; orchestrator reads only the tail | path recorded in the tasklist row |

Consistency rule: a ledger that exists only in the model's head does not count. If a
session ends and no ledger file exists for a substantive run, that is a fire failure,
reported honestly as skipped in the gates line -- never retro-claimed.

## 24. ORDERING IN THE ALWAYS-FIRE CHAIN

1. Fires AFTER godmode/promptgod (which establish the task) and after subagents
   (which size the fan-out this skill accounts for).
2. Fires BEFORE the turn's work: accounting precedes spending.
3. Alongside session-monitor-auto: this skill provides fill numbers and the
   state_file path; the monitor consumes them for stall/recovery decisions.
4. Outputs feed honestscore (ledger present, thresholds honored) and the goal loop
   (budget signal inside the stop-condition math).

If future godmode versions trim the chain (the RGAP2 structural note: promote the
missing primitives, demote mechanical checks into hooks), this skill survives as one
of the two primitives worth always-fire -- a budget line and a state_file line per
turn -- with the mechanical parts living in hooks.

## 25. WHEN *NOT* TO FIRE (honest scope limits)

- Pure conversational turns with no tool use and no file writes: no ledger row
  required (a substantive-run marker was never set).
- Sessions already inside a harness with real token accounting (SDK
  total_cost_usd present): do not duplicate -- record the measured number, skip the
  estimator. Running both violates the "measured replaces estimated, never both"
  rule and doubles the bookkeeping it was meant to prevent.
- Pure chat/voice-dictation turns where the user asked a one-line question: the
  chain's own tiering rule ("match effort to the task") applies; fire only if the
  session is already past WARN, in which case even the trivial answer carries a
  one-line budget note.

## 26. COST UNITS QUICK REFERENCE

| Signal | Source | Unit | When it exists |
|---|---|---|---|
| est_cum_tokens | this skill's estimate | tokens | always (fallback) |
| fill% | derived | percent | always |
| credits_remaining | context get-usage | credits | when the context MCP is wired |
| total_cost_usd | Claude Agent SDK sessions | USD | SDK-based runs |
| provider spend line | LLM gateway / litellm virtual keys | USD | when a gateway is in front |

Declaration discipline (pairs with skill-autonomous-run-budget): a run that will
spend money declares its budget in YAML at start; this skill's beat carries the
spend line; the breaker trips at 80%. The end-of-run budget report cites the ledger,
so the two skills close each other's loop and neither can claim "budget was fine"
without a row to point at.
