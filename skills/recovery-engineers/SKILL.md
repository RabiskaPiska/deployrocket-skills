---
name: recovery-engineers
description: Post-execution recovery engine. When a spawned subagent crew finishes (or dies) and the exported outputs do not match the expected outcome, this skill spawns 2 AI-engineer seats (default 2, configurable 1) that diff actual vs expected outputs, run ideas/deep-thinker/deep-research-plus loops to find the real root cause, design corrective prompts for ONLY the failed seats, launch the repair wave, and monitor it to the goal (pairs with session-monitor-auto). ABORT CASE - 15 identical-error loops means the task aborts with a short table summary plus a small MD report of why it failed, what was tried, and the last 3 attempts. alwaysFire - runs on any failed, partial, or diverged subagent wave before the orchestrator declares anything done.
alwaysFire: true
---

# recovery-engineers - the post-execution recovery engine

You are running **recovery-engineers**. A wave of spawned subagents was dispatched against
a goal. The wave is now in one of four states: some seats failed, some seats delivered
something that does not match the expected outcome, some seats diverged, or the whole run
is done but the merged artifact fails verification. Your job is to convert that state into
a REPAIR WAVE - a small, surgical, preflight-validated set of corrective prompts dispatched
to exactly the seats that failed - and monitor it until the goal is actually reached.

**What this skill is NOT:** a re-run button. Re-running a failed spawn with the same
prompt produces the same failure; the measured cost of blind respawn is the most expensive
failure class in the campaign log. Every repair is designed, not repeated.

---

## 0) HARD RULES (survive compaction - never violate even if everything else is lost)

1. **Default crew: 2 AI-engineer seats. Configurable to 1 when budget is tight. Never 3+**
   - recovery is a focused design task, not a fan-out; more seats re-correlate and duplicate
     diagnoses. 2 seats see the failure from two angles and must reconcile; 1 seat is the
     budget fallback with the dedup rule extra-strict (Section 6).
2. **Repair the failed seats only.** A repair wave never re-dispatches a seat that delivered
   its mandate. If you are not sure a seat failed, verify its artifacts first (Section 6) -
   the dedup rule exists because "probably failed" has destroyed verified work.
3. **Root cause before corrective prompt.** No corrective prompt without a stated root
   cause and the evidence that discriminates it (ideas Step 2). "Try again with more
   effort" is not a root cause and is a banned corrective prompt.
4. **Every corrective prompt passes prompt-preflight before its repair spawn.** A repair
   prompt that re-introduces a Gate A/B defect (unresolved ${VAR}, cached creds, missing
   cursor loop, gates line not last) re-dispatches the original failure.
5. **The 15-loop abort is arithmetic, not a judgment call.** Count identical-error loops
   per task; at 15, abort: deliver the summary table + the abort MD report. Do not run 16.
6. **Monitor to the goal, not to the dispatch.** A repair wave that "completed" without
   meeting the goal criteria is an unfinished run; the watchdog stays armed until the goal
   criteria are met or the abort case fires (pairs with session-monitor-auto).
7. **Durable findings go to the brain the same turn** (symptom -> cause -> fix). A recovery
   that fixes the run but not the runbook guarantees the next campaign pays again.
8. **Never trust self-reported recovery.** A seat that says "fixed" with no artifact is a
   lying completion report; classify it as still-failed and act on it.

---

## 1) WHEN IT FIRES

Fires automatically when ANY of these is true after a subagent wave completes (or times out):

- The orchestrator's merge finds the deliverable does not satisfy the goal criteria.
- One or more child final-message JSONs report `status` other than delivered/success, or
  the JSON is missing/blockers non-empty.
- A child delivered an artifact that exists but fails its own acceptance test.
- The watchdog (session-monitor-auto) fired a correction and the re-check still fails.
- A wave timed out with no final message (silent finisher).
- The goal loop (goal skill) reports an iteration that produced no net-new side-effects.

**Skipped paths (report as `skipped: why`, never silently):**

- `skipped: no spawn tool` - runtime cannot spawn; the recovery runs in-process: the same
  diagnose -> corrective-prompt -> re-execute loop executed serially by the orchestrator,
  honestly labeled.
- `skipped: goal already met` - the wave's "failure" was a false alarm (verify first!).
- `skipped: budget` - no spawn budget left; deliver the abort-style summary report instead
  of a repair wave, and say exactly what a fresh session must run to resume.

---

## 2) THE TWO-SEAT MODEL (default 2, configurable 1)

Each recovery seat is an AI-engineer with a distinct role. They read the same inputs and
are REQUIRED to diverge before comparing.

| Seat | Role | Owns | Must NOT |
|------|------|------|----------|
| RE-A | reproduce + localize: run the failed seat's steps against the exported outputs, find WHERE the pipeline diverges (step, input, runtime, or expectation) | the diagnosis + the corrective prompt for the seats whose failure is mechanical/config/spawn-shaped | rewriting artifacts directly |
| RE-B | re-frame: challenge RE-A's root cause with ideas/deep-thinker; when the failure depends on external facts (API shapes, versions, limits), run deep-research-plus with current-year sources and settle it | the corrective prompts for the seats whose failure is semantic/approach-shaped; the adversarial check on RE-A's diagnosis | re-diagnosing mechanical failures RE-A already proved |

**Why 2 and not 1:** a single recovery engineer reuses the same reasoning that produced the
original failure (the same reasoning that produced the error produces the same review).
Two seats with different tools (reproduction vs research) produce competing hypotheses,
and the cheaper discriminating test picks. **Why not more:** two seats already give the
de-correlation; a third duplicate-scope seat violates seat-disjointness and doubles the
token bill for no new evidence axis.

**1-seat mode (budget-tight, `RECOVERY_SEATS=1` or orchestrator directive):** one seat
performs both roles SEQUENTIALLY (reproduce first, research second) and must run the
discriminating test before writing any corrective prompt. The trade-off is honestly
reported: single-seat recovery loses the de-correlated diagnosis, so its corrective prompts
get the strictest preflight pass and the abort threshold drops from 15 to 10 identical
-error loops.

**Seat configuration (environment, defaults):**

```
RECOVERY_SEATS=2        # 1 or 2; 3+ is forbidden (duplicate scope, no added evidence)
REPAIR_WAVE_MAX=2       # corrective waves per failed seat before escalation to 1-way-door review
IDENTICAL_ERROR_ABORT=15  # 15 identical-error loops -> abort the task (10 if 1-seat mode)
REPAIR_RESUME=resume    # resume | fresh  (default: resume the failed seat's session)
```

---

## 3) INPUTS - the exported outputs vs the expected outcome

Recovery starts from EVIDENCE, never from memory of what "should" have happened. Required
inputs, gathered before any diagnosis:

1. **The exported outputs of the spawned subagents:** for each seat - its final message
   (including the fenced JSON status block), its deliverable file(s) on disk, and its
   session record (messages via cursor-paginated REST; ALL pages - the page-1-only
   undercount is a measured false-measurement class).
2. **The expected outcome:** the goal statement + the tasklist (ID / IN / OUT / deliverable
   / wave) + each mandate's acceptance test. If the acceptance test was never written, the
   first recovery act is to write it from the mandate - recovery without a target is noise.
3. **The dispatch records:** the exact prompts (hashes) and spawn calls, so the diagnosis
   can distinguish "the prompt was wrong" from "the runtime ate it" from "the seat went
   off-mandate".
4. **The preflight verdict records** for the original wave (if preflight ran; if it did
   not, run it now on the original prompts - the defect may predate dispatch).
5. **The runlog/watchdog beats** (if any) - flat-beat evidence, stall classifications,
   injected corrections and whether they landed.

Normalization step: build one table per seat before diagnosing anything:

```
| seat | mandate (1 line) | expected artifact + acceptance test | actual artifact | first divergence point | still-alive? |
```

Every row must cite a real side-effect (file path + size, exit code, HTTP status). A row
that says "delivered" with no artifact is recorded as FAILED (lying completion).

---

## 4) PHASE 1 - TRIAGE: classify each failure (do not fix yet)

Classify every failed seat into exactly one class. The class decides the repair route.
Misclassification here is the #1 wasted repair wave.

| Class | Signature | First hypothesis | Repair route |
|-------|-----------|------------------|--------------|
| C1 spawn-shape | child ran on the wrong model, or answered from the wrong context, or the session exists but no work happened | provider.no-route / model-ref shape / sessionID misuse (silent - the runtime never errors) | fix the SPAWN CALL, not the prompt; re-dispatch with the corrected payload |
| C2 mandate-gap | the child worked hard and produced a coherent artifact that answers a slightly different question | the mandate was ambiguous or under-scoped; the child optimized the wrong thing | rewrite the mandate (acceptance test first), resume the seat with the corrected contract |
| C3 runtime-failure | non-zero exit, parser error, encoding garbage, 401/403/timeout, empty pagination tail | a Gate B runtime known-failure (em-dash bytes, ${VAR}, cached password, missing cursor loop, exit code through a pipe) | apply the known fix from the registry; do NOT re-derive it |
| C4 wrong-direction | output is active and growing but diverges from the mandate | the child reinterpreted its OUT, or invented scope | steer via session resume with a narrowed IN/OUT; escalate to fresh spawn if it repeats |
| C5 upstream-blocked | the child's own blockers say "no API key / quota / endpoint dead" | a manual input or a dead dependency - not a prompt defect | do NOT respawn; surface the single item only the owner can supply |
| C6 goal-fault | every seat delivered exactly its mandate, but the goal is still unmet | the DECOMPOSITION was wrong (a gap between mandates) | back to the orchestrator for re-decomposition; repairs are wasted |

Discriminating evidence per class, cheapest first:

- C1: read the child's session record - model string, spawn input, injected DENY events.
- C2: diff the child's deliverable against the mandate sentence; if the artifact satisfies
      a plausible-but-different reading of the mandate, it is C2, not C3.
- C3: re-run the exact failed command unpiped; check the exit code at the source.
- C4: diff the artifact's actual claims against the mandate's IN/OUT lines.
- C5: read the child's blockers verbatim; check whether the blocker is owner-only (API key,
      login) - if yes, STOP and surface it; respawning cannot fix it.
- C6: coverage check - the union of all IN-sets vs the goal. A hole here indicts the
      tasklist, not the seats.

**The dedup gate BEFORE any repair (mandatory):** for every seat marked failed, verify its
surviving artifacts first (Section 6). A "failed" seat whose artifact actually satisfies
the acceptance test is not failed - the checker was broken. Fix the checker, not the seat.

---

## 5) PHASE 2 - ROOT-CAUSE LOOPS (the ideas / deep-thinker / deep-research-plus engine)

The two seats run their loops in parallel but converge on ONE diagnosis before any
corrective prompt is written.

### RE-A - the reproduction loop

1. Reproduce the failure from the artifacts (run the failed step, unpiped, and read the
   real exit code - never the piped one). A failure that does not reproduce from the
   exported outputs is an environment/credential problem (C5), not a prompt problem.
2. State 2-3 competing root-cause hypotheses with rough confidence (ideas Step 2). At
   least one must be "the original prompt/spec was wrong" and one must be "the runtime
   behaved per known registry class" - if both are excluded, escalate to C6.
3. Run the cheapest test that SEPARATES the hypotheses. Grep the brain runbooks first
   (ideas Step 0): a known root cause with a known fix is candidate #1.

### RE-B - the research loop (fires when external facts matter)

1. If the failure depends on anything current - an API shape, a version, a rate limit,
   a pricing change, a policy - run deep-research-plus / deep-thinker with CURRENT-YEAR
   sources. Last-known-good from memory is a banned input for a repair decision.
2. Convergence rule: stop researching when two independent sources agree or one
   authoritative primary source settles it; log the source for each load-bearing fact.
3. Deliverable: a one-paragraph refutation or confirmation of RE-A's diagnosis, with the
   citation trail. RE-B does NOT design its own competing repair until it has either
   refuted RE-A's diagnosis or run out of separating tests.

### Convergence gate (before Phase 3)

- Both seats agree on the root cause, OR one seat's diagnosis survived the other's
  cheapest discriminating test. A split with no test run is NOT a convergence - run the
  test or record the disagreement in the abort report honestly.
- The root cause is written as: symptom -> cause -> evidence -> fix-class (spawn-shape /
  runtime / mandate / upstream / decomposition), one line each, appended to the runlog.
- If the root cause is a NEW failure class (not in the prompt-preflight registry), the
  registry gets the append THIS TURN (update rule) and the brain note is written.

---

## 6) THE DEDUP RULE - verify surviving artifacts before respawn

Respawning a seat whose artifacts are good destroys verified work and re-runs cost. Before
ANY repair spawn:

1. **Inventory the surviving artifacts** of the whole wave: files with sizes + hashes,
   passing tests with exit codes, merged fragments, signed evidence bundles.
2. **Classify each artifact:** KEEP (satisfies its acceptance test - verified, not
   assumed), REPAIRABLE (partially correct; the repair extends it), DEAD (wrong or
   corrupt; regenerate).
3. **The corrective prompt states the classification verbatim:** which artifact paths the
   repair seat must NOT touch, which fragments it must read before writing, and which
   outputs are expected to change. A repair spawn that cannot name the surviving
   artifacts it must preserve is rejected by preflight (repair addendum).
4. **Cross-seat dedup:** if the failure is shared by N seats (same root cause), ONE
   corrective prompt pattern is designed and specialized per seat - never N independently
   designed repairs (the per-finding-fixer anti-pattern).

Budget-tight variant (1-seat mode): the dedup pass is the FIRST action, before diagnosis,
because a single seat has no partner to catch a destroyed artifact.

---

## 7) PHASE 3 - THE REPAIR-WAVE SCHEMA (corrective prompt format)

Every corrective prompt follows this schema, verbatim in structure. Preflight (Section 16
of prompt-preflight) enforces it; this is the reference shape.

```
REPAIR DISPATCH - seat <original-seat-id> - repair wave <N>

ORIGINAL DEFECT (one sentence, evidence-linked):
  <what was expected> / <what was actually produced> / <the exact symptom:
  exit code, missing file, wrong model string, gate-line absence, ...>

ROOT CAUSE (one line, from the Phase-2 agreement):
  <e.g. "spawn payload omitted providerID; runtime silently routed to
  experiential/glm-5.3-flash-uncensored instead of the requested model">

ALREADY TRIED (do not repeat):
  1. <attempt> -> <result + evidence path>
  2. <attempt> -> <result + evidence + why it failed>
  3. <attempt> -> <result + evidence>

SURVIVING ARTIFACTS (do not regenerate, do not delete):
  - <path> (<sha256[:16]>) - KEEP, verified <how>
  - <path> - REPAIRABLE: read before writing, extend in place

YOUR SCOPE (IN):  <only the failing sub-scope>
OUT (do not touch): <every seat/scope that passed, named individually>

CORRECTED INSTRUCTIONS (the actual fix, quoted, not described):
  1. <step - imperative, terminal-testable>
  2. <step>
ACCEPTANCE TEST (this is how done is judged):
  <exact command / observable + expected output, e.g. "python3 proof.py --only T03
  exits 0 and results-vps1.json gains a T03 row with fires==21">

RUNTIME RULES (quote the ones the original violated):
  <e.g. "model ref: abliteratedllm/glm-5.3-flash-abliterated, exact shape">
  <e.g. "read the serve password from the freshest /tmp/oc-serve*.log AT RUNTIME">

LOOP BOUND: <abort criterion - identical-error count + what to do at it>
REPORT FORMAT: fenced JSON {"status","deliverable_path","findings","blockers",
  "files_touched"} - gates line as the LAST line of the final message.
```

Dispatch rules for the repair wave:

- **Resume vs fresh:** same scope, deeper fix -> `sessionID` resume of the failed seat
  (context intact). Repeated failure after 1 resume -> fresh spawn on the most capable
  available model, briefed with "a prior implementer attempted this; read the runlog for
  what was tried." Every spawn or resume counts against the wave budget.
- **One dispatch carries the complete fix list** for that seat - never one repair spawn
  per finding.
- **The repair wave is a later wave:** worker seats never get repaired mid-wave by a
  sibling; the orchestrator (or this skill acting as the orchestrator's arm) owns dispatch.
- **Repair prompts pass prompt-preflight** before dispatch, no exceptions (a repair wave
  that skips preflight re-dispatches the same defect - the measured #1 repair failure).
- **Scope guard:** the repair seat is told the goal, the fix, and the bound - nothing else.
  Context hygiene: no credentials inside the mandate (the runtime-read rule covers auth).

---

## 8) PHASE 4 - MONITOR TO GOAL (pairs with session-monitor-auto)

The repair wave is monitored under the session-monitor-auto beat protocol; this skill
supplies the beats' GOAL side:

- Every beat verifies each repair seat's real side-effect: a growing artifact, a new file,
  a passing check's exit code - never "still running".
- The beat table's ETA column carries the repair seat's remaining loop budget, so a seat
  approaching the abort horizon is visible before it burns it.
- After ANY correction injection: forced 2-minute beat (per the watchdog's rule).
- Two flat beats on a repair seat -> the watchdog correction fires; if the correction
  does not land, the seat is killed under the watchdog's hard recovery and its scope is
  either absorbed or re-dispatched with a fresh corrective prompt (which re-passes
  preflight, because the failed seat's diagnosis is now richer).

**Completion condition** (handback): the repaired seats' artifacts satisfy their
acceptance tests AND the re-merged deliverable passes the orchestrator's verification
pass AND honestscore meets the bar the goal declared (>= 9.0 docs / >= 9.5 code).
Only then does the recovery loop stop. "Repair seats all returned" is not the condition;
the goal is.

---

## 9) THE ABORT CASE - 15 identical-error loops

Counted per task (not per seat): the SAME error signature - same message class, same
failing step, no state change - recurring across attempts. Two different errors
alternating is NOT an identical-error loop (that is churn, handled by escalation below);
an identical error means the corrective prompt is not reaching the defect.

**At the 15th identical-error loop:**

1. STOP. No 16th attempt, no "one more angle". The abort is mechanical.
2. Save all partials to the run's out/ directory (path recorded).
3. Deliver, in this order:
   - a SHORT TABLE SUMMARY (in the final message - the user-visible artifact):

   ```
   | seat | mandate | status | loops | last side-effect |
   |------|---------|--------|-------|------------------|
   | R1   | ...     | FAILED x15 (same error) | 15 | results.json unchanged since 14:57 |
   | R2   | ...     | DELIVERED | - | report-vps2.md 3885B, tests 2/2 |
   ```

   - a SMALL MD REPORT (abort-report-<task>.md) with exactly three sections:
     1. **Why it failed** - the one-line root cause (or the honest "unknown - here is
        the closest discriminating evidence we have").
     2. **What was tried** - the corrective prompts, in order, each with its evidence.
     3. **The last 3 attempts** - verbatim prompts (or paths) + verbatim outcomes.
4. The abort report lands in the brain runbook the same turn (symptom -> cause -> fix, or
   "cause unknown, evidence: ...") so the NEXT attempt does not start from zero.
5. NEVER fake completion on an abort. The abort message states: what was completed (with
   evidence), what was not, and the exact resume steps for a fresh session.

**Escalation before abort (only if the loop count is below 15):** after 2 corrective
waves on the same seat with the same failure, the 2-seat crew reconvenes: RE-A reproduces
the failure fresh (clean context), RE-B researches whether the fix needs a different
approach entirely (ideas full procedure, N >= 5 options). If the cheapest discriminating
test says the failure is upstream (owner-only input, dead endpoint, quota wall), the loop
stops and the single blocking item is surfaced - never a respawn against a closed door.

---

## 10) DELIVERABLES OF A RECOVERY RUN

| # | Deliverable | Form |
|---|-------------|------|
| 1 | Triage table (Section 3 shape) | in the runlog + final message |
| 2 | Root-cause record (symptom -> cause -> evidence -> fix-class) | in the runlog + brain note |
| 3 | Corrective prompt(s), preflight-passed | files under the run's out/ dir, hash-recorded |
| 4 | Repair-wave beat log | watchdog beats.md (append-only) |
| 5 | Re-merged deliverable + verification verdict | the original deliverable path, re-verified |
| 6 | Goal-completion report or abort report | runlog final entry; abort => the small MD report |
| 7 | Brain updates (new failure classes -> registry + runbook) | same turn, date-stamped |

Final-message contract (this skill, to the parent): fenced JSON with all keys present -
`{"status": "recovered|partial|aborted", "seats_repaired": N, "seats_killed": N,
"loops_used": N, "deliverable_path": "...", "findings": [], "blockers": [],
"abort_report": "path-or-null"}`.

---

## 11) WORKED EXAMPLE - a repair wave end to end

**Setup:** goal = "all 10 campaign tasks produce terminal PASS on both VPSes". Wave 7
dispatched 4 shard seats. Outcomes: T01-T05 fine on vps1; T07 PASS; T03/T04/T10 fail on
both boxes with the same signature: `21 fires, gate=Y, then relay-kill - session terminated
post-work`. Watchdog beat 21 recorded the pattern.

**Triage (Phase 1):**

- Classification: C5-adjacent first (is the relay kill an infra outage?), then C3.
- Dedup check on surviving artifacts: shard s3's results JSON is complete and verified ->
  KEEP; the two dead seats' partials are REPAIRABLE but stale (12h old) -> the repair
  prompt orders a re-run, not a patch.

**Root cause (Phase 2, 2 seats):**

- RE-A reproduces unpiped: the relay kills the session AFTER the work completes (fires
  already recorded); cause is a transport-layer idle timeout, not the task logic.
- RE-B researches the serve's relay/session config for the current build (current-year
  source), confirms the timeout is a server-side knob, and cites the config line.

**Corrective prompt (schema-conformant):** defect = relay-kill after work; root cause =
server idle timeout; tried-list = 2 prior direct re-runs; scope = tasks T03/T04/T10 only;
corrected instruction = "batch tasks so each session stays under the relay's idle window;
poll every 30s to keep the session warm; verify terminal outcome via cursor-paginated
message read." Passed preflight (A3 cursor loop present, B4 credential rule quoted, A4
gates last).

**Repair wave + monitor:** one fresh seat (2nd attempt for this scope, resumed once
already -> fresh spawn rule), background, watchdog beats every 3 min. T03 lands at loop
+1 (PASS, 21 fires, gate=Y); T04/T10 land next beats. Union reaches 10/10; merge
re-verified; honestscore 9.0. Loop stops truthfully.

**What made the difference:** the dedup rule prevented a full re-run (s3's verified
results were kept); the abort criterion was armed BEFORE the wave started (so "grinding"
was visible as a countdown, not a surprise); the corrective prompt quoted the corrected
runtime rules explicitly.

---

## 12) ANTI-PATTERNS (automatic defects)

- Re-running the original prompt unchanged and calling it a repair (the #1 failure).
- Repairing ALL seats because it feels thorough - failed seats only; passing seats are
  protected by the dedup rule.
- A corrective prompt with no root cause, no tried-list, no surviving-artifacts list.
- Researching (deep-thinker) what a local grep can settle in seconds - cheap discriminating
  test first, always.
- Declaring recovery "done" when the repair seats returned, without re-running the merged
  deliverable's acceptance test.
- Silent kills: a repair seat that died gets a runlog line and a scope decision, never a
  shrug.
- Respawning past the abort counter because "it's about to converge" - 15 means 15.
- One corrective prompt per finding (per-finding-fixer anti-pattern) - one dispatch, all
  findings, per seat.
- Skipping preflight on repair prompts because "the original prompt was fine" - the
  original failure may BE a preflight-class defect.

---

## 13) QUICK REFERENCE CARD

```
RECOVERY TRIGGER: wave done + expected != actual (or timeout, or diverged child)
CREW: 2 AI-engineer seats (RE-A reproduce/localize, RE-B research/challenge)
      1 seat only if budget-tight -> then dedup rule FIRST, abort at 10 loops
INPUTS: exported child outputs + expected outcome + original prompts + preflight records
PHASE 1 triage: classify C1 spawn-shape / C2 mandate / C3 runtime / C4 divergence /
                C5 upstream-blocker / C6 decomposition-gap  -> NO fixes yet
DEDUP GATE: verify surviving artifacts BEFORE any respawn (KEEP/REPAIRABLE/DEAD)
PHASE 2 root cause: reproduce unpiped -> 2-3 hypotheses -> cheapest separating test
                    (RE-B runs current-year research when external facts matter)
PHASE 3 repair: corrective prompts per the schema, preflight-passed, ONE dispatch per
                seat with the complete fix list; resume same-scope, fresh after 1 fail
PHASE 4 monitor: watchdog beats vs the GOAL; flat 2 beats -> correct; goal or abort
ABORT: 15 identical-error loops -> save partials, short table + small MD report
       (why / what was tried / last 3 attempts) -> brain note same turn
DONE: acceptance tests pass + merge re-verified + honestscore bar met. Nothing less.
```

---

## 14) PER-CLASS REPAIR PLAYBOOK (triage classes C1-C6, full procedures)

Each class below expands its triage row (Section 8) into the operational procedure. The
playbook is ordered by how cheap the diagnosis is - run top-down, stop at the first class
whose evidence fits. Skipping the cheap classes to "think about it" is the measured
token-burn failure; guessing a class without evidence is the second.

### C1 - spawn-shape failures (the runtime never ran what you think it ran)

**Signature:** the child answered, but on the wrong model, or its session shows a spawn
input that does not match the mandate, or the session exists with near-zero work.

1. Pull the child's session record (cursor-paginated message read; ALL pages). Compare the
   recorded model string against the spawn call's requested ref, exact string.
2. Diff the spawn payload actually recorded against the prompt text you authored. The two
   disagree more often than either disagrees with intent.
3. Repair class: fix the SPAWN CALL (providerID/modelID shape, sessionID absence/presence,
   background flag), not the prompt. A prompt edit here is a no-op.
4. Re-dispatch rule: fresh spawn with the corrected payload; the old session is evidence,
   not a resume target (its context is poisoned by the wrong-model run).

### C2 - mandate ambiguity (the seat did coherent work on a different question)

1. Diff the artifact against the mandate line by line. Mark each artifact element:
   in-mandate / off-mandate / invented.
2. If the artifact is a coherent answer to a DIFFERENT question, the mandate was ambiguous
   - the defect is in the prompt, not the seat. Rewrite the mandate with the acceptance
   test FIRST, then resume the seat with the corrected contract (its context is an asset).
3. If the mandate was fine but the seat restated its own OUT, that is C4 - do not blur the
   two classes; C2 repairs by clarifying, C4 by steering or replacing.

### C3 - runtime failures (registry classes; known fixes, no re-derivation)

1. Identify the exact error signature (unpiped re-run for the true exit code).
2. Look it up in the prompt-preflight runtime registry (its Section 12) and in the brain
   root-cause notes BEFORE hypothesizing - a known fix is candidate #1; a known failed
   fix is a banned option.
3. Apply the registry fix. If the registry lacks the signature, you are authoring a new
   registry row (the update rule) - the repair wave may proceed with the fix, but the
   append happens THIS TURN, not after the campaign.
4. Regression check the fix against the checklist item it came from (re-introduce the
   defect mentally: would the fixed prompt have failed the original way? If the fix does
   not fail on the original defect, it is an inert guard - rewrite it until it would).

### C4 - semantic divergence (active output, wrong direction)

1. Do NOT kill immediately. One narrowed resume: restate IN and OUT in imperative form,
   list the sibling scopes with "do not cover these", name the divergence in one sentence.
2. If the resumed seat diverges again (same direction), kill it: fresh seat, most capable
   model, brief carries "a prior implementer attempted this; read the report file for what
   was tried", and the mandate is re-derived from the acceptance test, not from the old
   prose.
3. Log both attempts in the runlog - a divergence that survives one steering attempt is
   evidence the mandate itself was under-specified (upgrade the class to C2/C6).

### C5 - upstream blocker (only the owner can unblock; respawning is sabotage)

1. Verify the blocker is real: cite the exact failing call, its response code/body.
2. Classify the blocker: credential (API key / login / OAuth approval), a paid action
   (top-up / subscription / licence), or a registration only the owner can perform.
3. Do NOT respawn, do NOT substitute a worse tool silently. Surface the single blocking
   item, state precisely what it unlocks, and do every other part of the goal first.
4. The abort-style report applies: what completed (evidence), what is blocked (the one
   item), what a fresh session must do first when the item arrives.

### C6 - decomposition gap (every seat delivered; the goal is still unmet)

1. Re-run the coverage check against the tasklist: the union of IN-sets vs the goal's
   requirements. Name the uncovered requirement in one line.
2. This is an ORCHESTRATOR defect, not a seat defect. No corrective prompt to a seat can
   fix it. The recovery output here is a scope proposal: the missing sub-scope, the seats
   it borders, and the handoff edges - back to the orchestrator (or the goal loop), never
   a solo heroic repair by a recovery seat.
3. Budget note: a C6 found late has usually already eaten a repair wave. Record the wave
   as spent in the ledger (Section 16) - the honest bill, not a rounded one.

---

## 15) WORKED EXAMPLE 2 - the 15-loop abort, delivered

**Scenario:** repair seat R2 (verify-union-counts scope) fails with the identical signature
`KeyError: 'cursor'` on every attempt - 15 consecutive loops, same error, no state change.
The corrective prompts (2 waves) both correctly identified the cursor loop as the fix, but
the serve's message endpoint for THAT session returned a shape without the `pagination`
key (empty session edge case the schema never showed).

**Abort delivery, verbatim shape:**

Short table summary (in the recovery's final message to the parent):

```
| seat | mandate             | status            | loops | last side-effect                     |
|------|---------------------|-------------------|-------|--------------------------------------|
| R1   | reconcile results   | DELIVERED         | -     | results-vps1.json 15592B @14:57Z     |
| R2   | verify union counts | ABORTED at loop15 | 15/15 | results file unchanged since 14:27Z  |
| R3   | adversarial pass    | PENDING (blocked) | 0     | not spawned - R2 dependency          |
```

Small MD report (`ABORT-verify-union-2026-11-12.md`), exactly three parts:

1. **Why it failed:** the session-message endpoint returns `{data:[...], cursor:{...}}`
   except when the session has zero messages, where `pagination` is absent - the verify
   script indexed `pagination.cursor` unconditionally (line 37 of verify_union.py).
2. **What was tried:** (1) cursor-following rewrite, loop 1-5; (2) defensive `.get()`
   chain + retry, loop 6-9; (3) endpoint switch to the project-list fallback, loop 10-15.
   Each attempt's diff path + exit codes listed.
3. **Last 3 attempts** (verbatim prompts, or paths): `out/r2-attempt-13.md`,
   `out/r2-attempt-14.md`, `out/r2-attempt-15.md` with their one-line outcomes.

Plus the two standing obligations: partials saved (list of paths + hashes), brain note
appended THIS TURN (symptom -> cause -> fix) so the empty-session edge is never
rediscovered. The abort is a DELIVERABLE, not a failure to report.

---

## 16) WORKED EXAMPLE 3 - C5 upstream blocker (the discipline case)

**Setup:** three seats passed; one seat's task is "re-run the paid image pack against the
client's brand key". The seat's blocker: `402 USAGE_EXCEEDED - credits exhausted, next
refill <date>`.

Wrong recoveries (all observed in the wild):

- Re-dispatch the seat hoping the quota resets (it will not; 15 loops of the same 402).
- Silently switch to a free generator to "keep momentum" - violates the mandate (the
  client approved THAT generator's output class).
- Mark the whole task FAILED and abort the goal over one blocked sub-scope.

Correct recovery:

1. Classify C5 (upstream, owner-only input). No respawn of any seat.
2. Surface exactly one item to the owner: "the credit balance is exhausted; a top-up
   unlocks the remaining 3 renders." Do every other part first (the other seats' merges,
   the docs, the CONTINUE-HERE refresh) so the owner is blocked on exactly one thing.
3. Ledger the seat as `BLOCKED-OWNER` - not FAILED (it did nothing wrong), not KILLED
   (its scope is intact and resumes in one call once the input exists).
4. When the credit lands: resume the SAME session (context intact) - the resume is a
   continuation, not a new spawn, and does not re-run the dedup gate for the other seats.

The class matters because each wrong recovery has a distinct cost: respawn burns budget,
silent substitution ships the wrong deliverable class, and a false-FAIL poisons the goal
loop's progress arithmetic.

---

## 17) THE RECOVERY LEDGER (crash-resumable state file)

Long recoveries get killed mid-wave (context reset, box reboot, relay kill). The ledger
file makes the recovery resumable by ANY fresh session - the state lives in a file, never
in the recovering seat's memory.

Path: `<run-dir>/recovery-ledger.json`. Appended/updated every phase transition. Schema:

```json
{
  "kind": "recovery-engineers-ledger",
  "goal": "<one-line goal restatement>",
  "created_utc": "<iso8601>",
  "seats": [
    {
      "seat_id": "R2",
      "original_mandate": "<1 line>",
      "triage_class": "C3",
      "evidence": ["exit 1 on proof.py --selftest", "results.json missing T03 row"],
      "artifact_class": {"results-vps1.json": "KEEP", "verify_out/": "DEAD"},
      "corrective_prompt": "out/repair-R2-prompt.txt",
      "preflight": "PASS (sha c81a...)",
      "respawn": {"session_id": "ses_...", "kind": "fresh", "attempt": 2},
      "identical_error_loops": 0,
      "status": "REPAIRING"
    }
  ],
  "root_cause": "symptom -> cause -> evidence (one line each)",
  "repair_waves": [
    {"wave": 1, "dispatched_utc": "...", "seats": ["R2"], "verdict": "PASS-WITH-FIXES"}
  ],
  "abort_state": null,
  "handoff": "resume from phase 3; R2 corrective prompt is preflight-passed"
}
```

Rules:

- The ledger is the truth, not recollection - a recovery that lost its place re-dispatches
  completed repairs (the same expensive failure class as the orchestrator's lost place).
- `attempt` counts against the 2-retry rule per seat; attempt 3+ requires the fresh-seat
  upgrade path (most capable model + "a prior implementer attempted this" context).
- On abort, `abort_state` carries the loop counter, the error signature, and the 3-attempt
  list - the small MD report is generated FROM the ledger, never from memory.
- The ledger path is recorded in the runlog and in CONTINUE-HERE; a ledger that only
  exists in the session's memory is a ledger that does not exist.

---

## 16) ESCALATION MATRIX (when a recovery seat calls for help, and to whom)

| Trigger | Handoff | Deliverable expected back |
|---------|---------|---------------------------|
| Root cause needs >2 hypotheses separated by >1 test each | `ideas` (full path, N=5-8) | scored candidate table + chosen fix + kill-criteria |
| Failure depends on current-year external facts (API shape, version, pricing, policy) | `deep-thinker` / `deep-research-plus` | cited source report, convergence stated, uncertain items flagged |
| The corrective prompt keeps failing preflight on the SAME gate 2+ times | back to the prompt factory (`/prompt`) - the fault is upstream of recovery | regenerated prompt that passes preflight cleanly |
| Every seat delivered but the goal is unmet (C6) | the goal loop / orchestrator | a re-decomposition, not a repair |
| All classes cleared, deliverable re-merged | `honestscore` | MIN across dimensions + typed defect tickets |

Escalation is one-way per trigger: a recovery seat that escalates without exhausting the
cheaper local check first is spending the verify reserve on thinking it could have
grepped. The reverse - skipping research on an external-fact dispute because reasoning
FEELS sufficient - is the deeper failure; the current-year rule exists for exactly that.

---

## 17) BUDGET ACCOUNTING (the recovery run's own caps)

A recovery inherits the parent run's budget and adds its own; both are declared before
phase 1 and reported at the end:

- **Spawn cap:** every corrective spawn or fresh respawn counts against the wave/run
  budget; a `sessionID` resume is a continuation, not a new spawn. At the cap, remaining
  fixes are applied by the orchestrator from the logged findings - reported honestly.
- **Retry rule:** <= 2 corrective attempts per seat before the escalation matrix fires
  (the 15-loop abort counter bounds the WHOLE task; 2 retries bound a single seat).
- **LLM-call discipline:** triage and dedup are file/byte operations (zero LLM calls).
  The ideas/deep-thinker loops are reserved for phases 2-3 disputes, never for triage.
- **Wall-clock reserve:** the last ~25% of the recovery's window belongs to re-merge and
  re-verification; a repair wave launched inside the reserve must be the LAST one, or the
  run reports partials instead of compressing verification.
- **Beat cost:** watchdog beats write to beats.md, not to the LLM context; only the last
  3 beats are summarized in-context per the watchdog's own hygiene rule.

---

## 17) PREFLIGHT INTERPLAY - what recovery NEVER re-checks

`prompt-preflight` owns spawn-payload shape, runtime known-failures, slop, and seat
disjointness. Recovery owns outcomes, root causes, and the decision to repair. The
boundary rule prevents both gaps and double work:

- Recovery does NOT re-run the full preflight battery on a prompt that already has a
  recorded PASS verdict at its current hash - it verifies the hash matches instead.
- Recovery DOES force a full preflight re-run when the corrective prompt differs by more
  than the failing slot (any structural edit re-opens all gates - the hash record exists
  so this is detectable, not decidable by memory).
- A repair wave whose preflight verdict is REJECT is never dispatched "to see" - the wave
  waits for the fix. A wave launched on a REJECT verdict is a rule-8 violation (done
  claimed without verification, one level up).

---

## 18) FINAL REPORT FORMAT (recovery -> parent, every time)

```
RECOVERY REPORT - <run id>
Trigger: <one line - which seats, which mismatch>
Crew: RE-A + RE-B (or 1-seat mode: declared)
Triage: <seat> -> <class> -> <root cause one-liner>
Dedup: <artifacts kept: N> / <repaired: N> / <destroyed: 0 required - any DEAD call cited>
Repair waves: <N dispatched, verdicts, loops used per seat>
Outcome: <goal met + evidence | PARTIAL + what remains | ABORTED at loop 15 (report path)>
Budget: <spawns used / cap> <llm loops used> <beats monitored>
Brain: <root-cause note path written this turn, or "none new - existing note updated">
```

The report never says "recovered" without the re-merged deliverable's acceptance evidence,
and never reports an abort as a near-success: 15 loops of identical error is a task-level
stop, delivered as the table + the small MD report, with the owner's next manual step
named in one line if (and only if) the blocker is owner-class (C5).

---

## 19) APPENDIX A - the identical-error counter (deterministic, not memory-based)

The 15-loop abort is arithmetic, so the counter must be deterministic and persisted.
Rule: the same NORMALIZED error signature on consecutive attempts of the same task.
Normalization strips timestamps, session ids, and per-run paths; it keeps the error
class + the failing frame + the failing assertion.

```python
# abort_counter.py - persist per-seat signatures; wire into every repair attempt
import hashlib, json, pathlib, re

LEDGER = pathlib.Path("recovery-ledger.json")   # the crash-resume state file
ABORT_LIMIT = 15                                # 10 in 1-seat mode - set per run

def normalize(err: str) -> str:
    s = re.sub(r"\d{4}-\d{2}-\d{2}[T ][0-9:.]+Z?", "<ts>", err)
    s = re.sub(r"ses_[A-Za-z0-9]+", "<sid>", s)
    s = re.sub(r"[\w./\-]+\.(py|json|md)", "<path>", s)
    return s

def record(ledger: dict, seat: str, err: str) -> bool:  # True when abort fires
    sig = hashlib.sha256(normalize(err).encode()).hexdigest()[:16]
    st = ledger.setdefault(seat, {"last_sig": None, "streak": 0})
    st["streak"] = st["streak"] + 1 if st.get("last_sig") == sig else 1
    st["last_sig"] = sig
    json.dump(ledger, open("recovery-ledger.json", "w"))
    return st["streak"] >= ABORT_LIMIT                  # True -> Section 9 procedure
```

Counter rules (each prevents a measured failure):

- The signature is persisted in the recovery ledger, not in memory - a context reset
  that loses the counter has lost the abort discipline with it.
- "Identical" means CONSECUTIVE identical signatures, not 15 total attempts: error A ->
  B -> A -> A is churn between two distinct failures (escalate), not an identical-error
  loop. Default policy is consecutive; the ledger records which policy was armed.
- The counter is per task, not per seat, when the seats share one root cause: the same
  signature from 3 seats in one wave is 15 shared loops and ONE abort decision.

## 20) APPENDIX B - artifact-integrity check (the dedup rule, enforced)

Runs BEFORE corrective prompts are finalized - its output can change the repair scope:

```python
# dedup_integrity.py - verify KEEP artifacts against the ledger before respawn
import hashlib, json, sys
ledger = json.load(open("recovery-ledger.json"))  # artifacts: path -> {"sha16","class"}
bad = []
for path, meta in ledger["artifacts"].items():
    if meta["class"] == "DEAD":
        continue                                    # scheduled for replacement
    try:
        got = hashlib.sha256(open(path, "rb").read()).hexdigest()[:16]
    except OSError:
        bad.append((path, "MISSING")); continue
    if meta["class"] == "KEEP" and got != meta["sha16"]:
        bad.append((path, f"drift: {meta['sha16']} -> {got}"))
print("dedup-integrity:", ("FAIL " + str(bad)) if bad else "PASS")
sys.exit(1 if bad else 0)
```

Interpretation rules:

- A KEEP artifact whose hash drifted is a finding in its own right: something ran between
  the last verification and the repair - two writers were active, and the nominal failure
  may be a symptom of exactly that race (the campaign's duplicate-proof.py lesson).
- A MISSING keeper is a C3-class finding before anything else gets repaired.
- A drifted keeper may ADD a repair target to the wave - which is why this check runs
  before the corrective prompts are finalized, not after.

## 21) APPENDIX C - minimum monitoring loop (no-runtime variant)

Where the watchdog's full beat machinery is unavailable (no spawn tool, no Monitor tool),
the recovery still monitors - in-process, same rules, degraded honestly:

```
loop every ~3 min while repair seats are in flight:
  1. per seat: does a real side-effect exist since the last beat?
     (file mtime/size advanced, results count grew, exit code recorded)
  2. flat for 2 consecutive beats -> inject/steer the correction or absorb the scope
  3. append the beat to beats.md citing the side-effect - never the word "running"
  4. bounds: seat retry budget per beat; task abort via the 15-counter
exit: goal met + evidence | abort path (Section 9) | UNVERIFIED named per seat
```

The no-tool variant is acceptable ONLY when no spawn capability exists at all (the
subagents skill's degradation path). A spawn-capable runtime that skips monitoring is a
violation, not an optimization.

