---
name: goal
description: Goal-driven iteration engine - drive any declared goal to DONE by looping the full skill stack (ideas -> specialist agents -> subagents -> verification -> honestscore) until a compound stop condition fires. Use whenever the user declares a goal, target, or "keep going until X" - especially long autonomous pushes (48 hours to 10 days), loop-until-score runs (9.5/9.8), or any task needing unique ideas every iteration plus a 10-30 subagent crew.
alwaysFire: true
---

# goal - The Goal-Driven Iteration Engine

## 0. IDENTITY

goal is the loop engine. When the user declares a goal ("keep going until X",
"48 hours to 10 days", "loop until >= 9.5", "drive this to done"), this skill owns
the run: it declares the goal, locks a definition-of-done, iterates the full skill
stack, tracks every attempt in a fingerprinted ledger, schedules work in waves, and
stops ONLY on a compound stop condition (done, or abort criteria honestly fired).

The three laws:

1. NEVER trust self-reported done. A goal is DONE only when every DoD criterion has
   external evidence attached (command output, artifact, measurement).
2. NEVER repeat an attempt. Every iteration must carry a NEW idea fingerprint
   (Section 5); a repeated idea without new evidence is a stalled run, not progress.
3. NEVER run forever. The abort cap (Section 7) ends honestly what cannot end well.

## 1. TRIGGER

This skill fires when the user message contains a declared goal, target, or
stop-condition phrasing:

- "keep going until X", "loop until X", "get to X"
- "48 hours to ...", "10 days of ...", "until it hits N"
- "iterate until score >= N", "do not stop until ..."
- A numeric target with no deadline ("rank top 3", "9.5/10", "100% gate rate")
- An explicit "activate goal skill" / "@goal" reference

A task WITHOUT a declared end-state ("fix this bug") is an ordinary task - do the
work, do not spin up a goal loop. A goal loop exists for compound, long-horizon,
or quality-threshold objectives.

## 2. STEP 1 - PARSE THE GOAL DECLARATION

Extract, in writing, before any iteration:

```
GOAL:        <one sentence, the user's words cleaned up>
DOJ:         <what DONE means - observable state, not effort>
DO-NOT:      <explicit non-goals; scope the loop does not touch>
STOP-SCORE:  <honestscore threshold, or "n/a">
CAPS:        <max rounds / max wall-clock / max cost - declared or defaults>
EVIDENCE:    <how completion will be proven: the exact check(s)>
```

Defaults when the user declared none:

- CAPS: 20 rounds, 48h wall-clock, declared tool-call budget.
- STOP-SCORE: 9.0 for docs/sites, 9.5 for critical code (from honestscore).
- COMPOUND STOP: goal met AND honestscore >= threshold AND no open defect tickets.

If the goal is ambiguous (no observable DONE state), ask ONE clarifying question
before arming - this is a real manual input, not a permission-ask. If unanswerable,
construct the most defensible interpretation, state it in the ledger, and proceed.

## 3. STEP 2 - DEFINITION OF DONE (DoD)

DoD criteria are MEASURABLE statements, each with its falsifier:

```
DoD-1: gate-line rate >= 90% across 20 tasks | falsifier: independent re-scan
DoD-2: every results row has a terminal outcome recorded     | falsifier: grep
DoD-3: honestscore of the final report >= 9.0                 | falsifier: score block
```

Rules:
- Each criterion names the check that can come out the other way. "Looks good" is
  not a criterion.
- Count-based criteria state the denominator ("10/10", not "all").
- The DoD list is FROZEN once the loop starts; mid-run changes require re-declaring
  the goal (fingerprint change, Section 5) so the ledger never mixes runs.
- Include at least one regression criterion: the things that already worked still
  work.

## 4. STEP 3 - STATE FILE (survives context resets)

The loop's memory lives in files, never in chat context:

```
./goal-state/
  GOAL.md          # the parsed goal, DoD, caps, compound stop condition
  ledger.jsonl     # fingerprinted idea ledger (Section 5)
  tasklist.md      # the crew tasklist (subagents protocol, Section 7)
  beats.md         # monitor beats (session-monitor-auto writes these)
  CONTINUE-HERE.md # refresh at every milestone, not only at the end
```

Every round: update ledger, tasklist, and CONTINUE-HERE BEFORE context grows. A
context reset (or a fresh session on another machine) must be able to resume by
reading goal-state alone. If the run can survive `rm -rf` of your memory, the state
is correct.

## 5. STEP 4 - THE FINGERPRINTED IDEA LEDGER

Every iteration's approach is fingerprinted BEFORE execution:

```
fingerprint = hash(normalize(essence-of-approach))
```

- The fingerprint covers the APPROACH, not the wording: what will be tried, on what
  surface, by what mechanism. (Example: "add prune logic to sync script guarded by
  allowlist" and "sync script: allowlist-guarded prune" share a fingerprint.)
- ledger.jsonl is append-only. One JSON object per attempt:

```json
{"n": 7, "fp": "a1b2c3d4e5f60718", "idea": "one-line approach",
 "source": "ideas-round-3|user|subagent-S4", "wave": "3",
 "result": "pending|pass|fail|partial", "evidence": "path-or-command",
 "defects": ["ticket-ids"], "ts": "2026-11-12T14:03Z"}
```

Uniqueness rule: before executing an attempt, hash it. If the fingerprint exists in
the ledger with a terminal result (pass/fail) and NO new evidence has arrived since,
REJECT the attempt - go to ideas for a genuinely different approach. Retrying the
same idea is only legitimate when the prior attempt failed for an infra reason
(retry-after-timeout), and then the ledger entry records "retry, cause=infra".

The ledger is also the anti-loop proof: two attempts with the same fingerprint and
no state change between them = the run is looping = abort criteria candidate
(Section 8).

## 6. STEP 5 - WAVE SCHEDULING (the crew)

Each iteration decomposes the remaining work into disjoint mandates and schedules
waves per the subagents protocol (floor 5 / ceiling 30, 5-8 launches per message,
tasklist-first, depth-1). The goal engine's additions:

- Wave 0 (once): recon the current state; produce the baseline measurement the
  future rounds compare against. No baseline = no honest "improved".
- Every wave launch updates tasklist.md rows; every delivery updates the ledger.
- Seats never edit the goal state files; only the orchestrator writes ledger.jsonl
  (single-writer rule).
- An iteration ends when its wave evidence is merged and verified - not when the
  seats return.

## 7. STEP 6 - THE LOOP (one iteration, in order)

```
1. READ state (goal-state dir is the truth, not chat memory)
2. PICK the highest-leverage open gap (largest DoD gap with the cheapest falsifier)
3. IDEAS: if the last attempt failed, generate >= 3 distinct approaches; fingerprint
   each; reject ledger repeats; pick by evidence
4. TASKLIST: update rows for this round's crew
5. LAUNCH the wave (single message, 5-8 seats, disjoint mandates)
6. MONITOR: arm session-monitor-auto; beats every 3-5 min with real side-effects
7. MERGE: evidence wins conflicts; verify-after-merge (recompute claims)
8. VERIFY: run the round's falsifiers; attach outputs to the ledger entry
9. SCORE: honestscore on the round's deliverable -> score into ledger
10. CHECK STOP: compound stop condition (Section 9)
11. PERSIST: ledger, tasklist, CONTINUE-HERE, brain notes (root causes immediately)
```

The loop NEVER advances on assumption: step 8 requires real output (command exit
codes, files, counts). A round whose only product is chat text is not a round.

## 8. STOP CONDITIONS (compound)

The run stops when ALL of:

1. Every DoD criterion has external evidence attached (falsifier passed).
2. honestscore on the final deliverable >= threshold (or UNVERIFIED items resolved).
3. The regression check passed (nothing that worked before broke).
4. The adversarial pass found no surviving refutation.
5. Evidence persisted: ledger + CONTINUE-HERE + brain note written.

Partial credit does not exist: if 9 of 10 DoD rows pass and the 10th is UNVERIFIED,
the goal is not DONE. Either close it or declare the honest stop (Section 9).

## 8.1 THE ABORT CAP (stop honestly, not optimistically)

Abort when ANY of:

- ROUNDS: 2 full repair rounds with no net-new DoD progress (no new evidence for
  any criterion).
- BUDGET: declared round/wall-clock/cost cap hit.
- BLOCKER: the same blocker returned twice after correction attempts.
- INPUT: a missing manual input only the user can supply (credential, approval,
  payment) - stop and name exactly that one item (the user does only three kinds of
  things: supply a new credential, register something in their name, pay).
- INFRA: the environment is unrecoverable (service down, no path).

On abort:
1. Write ABORTED + criterion + evidence to GOAL.md and the ledger.
2. Save all partials (files, results, branch/worktree state).
3. Report: what was completed (with evidence), what is unfinished, the EXACT next
   steps and file locations for resume.
4. NEVER fake completion on an abort. An honest 7/10 with the blocker named beats
   a fabricated 10/10 - and the ledger makes the fabrication auditable anyway.


## 9. THE FINGERPRINT LEDGER AND /ideas (how failures feed back)

When a round fails (honestscore < threshold, or a falsifier fires):
1. Ticket each defect (honestscore ticket format).
2. Skill(ideas) on the highest-severity ticket: >= 3 DISTINCT approaches, scored.
3. The chosen approach's fingerprint goes into the ledger BEFORE the work starts
   (pre-commitment: you cannot retroactively claim novelty).
4. If ideas cannot produce a fingerprint that differs from all failed attempts, the
   approach space is exhausted -> escalate: change the frame (different tool,
   different scope, ask the user for the missing input) rather than repeat.

This is the anti-thrash mechanism: the fleet measured that loops without
fingerprinting repeat the same failed fix up to the cap and call it iteration.

## 10. HEARTBEAT (mandatory for any run > 5 min)

Arm session-monitor-auto the moment the loop starts. Beats verify REAL side-effects:

- ledger.jsonl grew (new attempt rows)
- tasklist rows transitioned (pending -> delivered)
- artifacts changed on disk (size/mtime/hash)
- process/session alive

Two flat beats = stall: diagnose (frozen process? dependency block? provider
outage?), recover (re-run / resume from the last ledger entry / re-dispatch the
seat), report the pulse. Never report "still running" without a side-effect.

## 11. CONTINUITY AND BRAIN PERSISTENCE

At every milestone (not only at the end):
1. Update CONTINUE-HERE-<date>-<goal-slug>.md in the backup dir: DONE / UNFINISHED /
   EXACT next steps + file locations - written so a fresh session on any machine
   can resume from the static resume prompt.
2. Append durable findings to the relevant brain workspace knowledge note
   (date-stamped, verified facts, rejected alternatives with reasons).
3. Root causes and gotchas go to the runbook the moment they are found
   (symptom -> cause -> fix), not at the end.
4. Durable changes get a GitHub record (commit + push + verify), per the standing
   directive.

## 12. WORKED EXAMPLE (compressed, real shape)

User goal: "keep iterating the fleet proof run until gate rate >= 90% on all 20
tasks, max 2 relaunch rounds."

- GOAL.md written; DoD: D1 gate-rate >= 90% measured by the independent scanner
  (not the stored flags); D2 >= 7 unique fires per task; D3 honestscore of the
  closure report >= 9.0; CAPS: 2 relaunch rounds.
- Baseline (wave 0): fresh harvest -> 40% strict rate, 17 flag mismatches. The gap
  is real; the loop is justified.
- Round 1: fingerprint "full-text-reparse-adj-tail" -> rebuild results with full
  final-text parsing -> 100% chat-gate, 40% strict. DoD1 still open. Ledger: pass.
- Round 2: fingerprint "restart-serve-plugin-reload" -> restart serves, re-run the
  12 non-canonical tasks in shards (3/box concurrent) -> strict 100%, mismatches 0.
- Verify-after-merge: independent recompute from the merged file = 20/20.
- Adversarial: provenance attack (were the passes post-restart?) survived.
- honestscore report 9.1 -> DONE. All evidence in runlogs + ledger; CONTINUE-HERE
  updated; relaunch rounds used: 0 of 2 (reported honestly).

## 13. INTEGRATION

- promptgod/godmode: the goal loop runs inside the godmode protocol; it inherits
  the gate-line obligation.
- subagents: crew protocol for every wave; the tasklist format is shared.
- session-monitor-auto: the watchdog for the loop; its beats are the loop's pulse.
- ideas: every failed round feeds it; every approach it produces is fingerprinted.
- honestscore: the per-iteration and final quality gate; its score is a stop
  condition input.
- skill-frozen-spec: for runs with a locked spec, the DoD freeze defers to the
  frozen spec's hash; drift = abort-and-redeclare, not silent scope creep.

## 14. GATE-LINE CONTRIBUTION

The goal engine's turn emits its state in the parent's gate line:

```
goal: fired (round N, ledger M ideas, DoD k/j met)
```

"fired" = real ledger writes and wave launches this turn. A goal declared in prose
with no state file is "skipped: why" in the gate line - the loop exists in files,
not in intention.

## 15. DO / DON'T

| DO | DON'T |
|---|---|
| Parse the goal into DoD with falsifiers | Treat the vibe of the request as the DoD |
| Fingerprint every attempt before running | Re-run a failed idea and call it new |
| Persist state every round | Trust chat context to survive |
| Score every round with honestscore | Claim DONE on a self-report |
| Abort at the cap, honestly | Loop forever on a dead approach |
| Escalate the missing-input item to the user | Ask permission for in-scope work |
| Verify side-effects every beat | Report "still running" on trust |

---

# APPENDIX A - DoD CRITERION CATALOG (with falsifiers)

Pick criteria from here; every one must be closable by a check that can fail.

| Type | Criterion template | Falsifier |
|---|---|---|
| GATE-RATE | "field X present on >= N% of M rows" | independent re-parse of the artifacts, not the stored flags |
| COUNT | "exactly K occurrences of Y" | recount with a different method (grep -c vs parser) |
| SCORE | "honestscore >= T on <artifact>" | the score block, adversarial pass attached |
| ARTIFACT | "<path> exists, parses, and satisfies <schema>" | validator run + exit code |
| REGRESSION | "previously passing check <name> still passes" | re-run it this round |
| PERF | "<metric> <= <bound> measured by <method>" | benchmark output, n >= 3 |
| SECURITY | "scan X reports no findings of class Y" | scanner output pasted |
| DOC | "every claim in <doc> cites a source fetched this run" | spot-check 3 random citations |
| BUDGET | "total cost <= C / total calls <= N" | ledger tally recomputed |

Anti-criteria (banned as DoD): "works well", "user satisfied", "production-ready"
(unmeasurable as written - decompose them or refuse them as DoD).

---

# APPENDIX B - LEDGER OPERATIONS (exact procedures)

## A.1 Append an attempt

Before executing the attempt, append the row with result="pending" and the
fingerprint. After evidence lands, REWRITE THAT ROW'S result field only via a new
append (the ledger is append-only; a result correction is a new row referencing the
old one). Never mutate history - the ledger's value is that it cannot be quietly
edited after a failure.

## A.2 Fingerprint normalization

Lowercase; strip punctuation; sort multi-part approaches into a canonical order;
hash the normalized string (any stable hash; 16 hex chars is enough). The test of a
good fingerprint: two sessions describing the same approach independently must
produce the same hash. If they would not, the fingerprint is over-fitted to wording
- loosen the normalization.

## A.3 The uniqueness check (every round)

```
fp = fingerprint(candidate approach)
matches = ledger rows where fp == candidate.fp AND result in (pass, fail)
if matches and no new state since those rows:
    REJECT -> ideas for another approach
if matches and the prior failure was infra (deadline, provider death, relay):
    allowed, mark retry-with-cause
```

## A.4 The tally (end of every round)

Recompute from the ledger, never from memory: rounds run, DoD criteria with
evidence (k/j), attempts by outcome, fingerprint collisions (same idea attempted
twice - should be zero), budget consumed. The tally line goes into the beat and
into CONTINUE-HERE. A number you cannot recompute from the ledger is not a number.

---

# APPENDIX C - LONG-HORIZON VARIANT (48 hours to 10 days)

For multi-day goals the loop adds:

- C1 MILESTONE SLICES: the DoD decomposes into milestones deliverable in one
  session; every milestone closes with evidence + CONTINUE-HERE refresh. Never
  hold a day's work only in chat.
- C2 BUDGET DECLARATION pre-flight: tokens, cost, tool-calls, wall-clock; circuit
  breakers armed at 80%. The budget report closes the run.
- C3 THE DAILY RHYTHM: each working day ends with either a milestone closed or an
  explicit WIP checkpoint (state + ledger + CONTINUE-HERE). A day that ends with
  state only in memory is a lost day waiting to happen.
- C4 CONVERGENCE WATCH: if 3 consecutive milestones each add < 10% toward the
  goal, the approach is plateauing - invoke ideas for a different frame before
  burning more rounds.
- C5 THE UNIQUE-IDEA RULE: across a 10-day run the ledger enforces that iteration
  7 does not repeat iteration 2. Collisions are visible in the tally; 2 collisions
  = forced approach change.

---

# APPENDIX D - RESUME PROTOCOL (fresh session / new machine / post-reset)

A resumed session runs, in order:

1. Read goal-state/GOAL.md (goal, DoD, caps).
2. Read ledger.jsonl tail: last 5 attempts, their outcomes, open defects.
3. Read tasklist.md: which rows are delivered / pending.
4. Read CONTINUE-HERE: the previous session's own next-steps list.
5. Recompute the DoD tally from evidence files (never from the previous session's
   claims).
6. Resume at the first unmet DoD criterion - not from the beginning, not from
   memory.

The resume test for the state files: a stranger (or a fresh model) can take DONE /
UNFINISHED / next-steps and continue without asking you anything. If they cannot,
the state file is the defect - fix it before continuing.

---

# APPENDIX E - FAILURE MODES OBSERVED IN REAL RUNS (and the countermeasure)

- F1 The deadline mirage: the harness's wall-clock killed sessions that completed
  later; the results table said FAIL while the evidence said PASS. Countermeasure:
  verdicts come from terminal outcomes + evidence (the ADJ rule), never from the
  harness's clock alone; timeouts are recorded as timeouts, distinct from quality
  failures.
- F2 The stub row: rows with duration 0.0 and fires copied from siblings. They
  poison both directions (a fake FAIL or a fake PASS). Countermeasure: any row with
  duration 0.0 or no session id is UNRESOLVED - re-run, never count on trust.
- F3 The self-acknowledging child: injected corrections get "acknowledged" with no
  action. Countermeasure: corrections demand execution, not acknowledgment; one
  re-inject, then absorb.
- F4 The byte-identical crew: unrelated tasks sharing identical fire lists. A copy-
  paste signature. Countermeasure: cross-row similarity check before trusting any
  aggregate.
- F5 The quiet scope creep: the goal drifted into adjacent work and the ledger
  never noticed (no fingerprint change, no DoD change). Countermeasure: DoD freeze;
  drift forces re-declaration.

## E.1 DO / DON'T (round level)

| DO | DON'T |
|---|---|
| Close a round with evidence in the ledger | Move on with the result only in chat |
| Record timeouts as timeouts | Let a harness kill masquerade as quality FAIL |
| Mark stub rows UNRESOLVED | Merge untrustworthy rows "either way" |
| Re-run before counting | Count on copied fires |

---

# APPENDIX F - BUDGET DECLARATION TEMPLATE (pre-flight, every goal run)

```yaml
run: <goal-slug>
declared: <timestamp>
budget:
  rounds: <N>              # iteration cap (default: until DoD, hard abort at cap)
  wall_clock: <e.g. 48h>
  cost: <currency cap, if provider calls are expected>
  tool_calls: <cap>
breakers:
  - at: 80% of any budget line
    action: pause, report, require explicit continuation
  - 2 flat beats (no side-effects) -> stall protocol
report:
  end_of_run: true   # budget report is part of DONE, not optional
```

A round that exceeds its declared budget mid-iteration finishes the current slice,
then stops and reports. Silent overruns are the failure this template exists for.

---

# APPENDIX G - MULTI-MACHINE RESUME (the fleet case)

When the goal spans machines (PC + VPS seats):

1. The state dir lives with the primary artifact; the ledger is the source of
   truth, not any one machine's chat.
2. Every wave that runs remotely writes its results to disk THERE and syncs the
   evidence back (files, not narration).
3. Cross-machine steps name their transport in the ledger (ssh/scp/relay) and the
   exact commands used - the next session re-derives nothing.
4. If the network path dies mid-run (inter-DC outage class), degrade with the
   fail-open rule, log the outage, and record which evidence is UNVERIFIED until a
   relay or retry lands it.

---

# APPENDIX H - THE ONE-PAGE OPERATING CARD

```
PARSE  goal -> GOAL.md (DoD + falsifiers + caps)  [ask ONE question only if the
       DONE state is genuinely unobservable]
BASE   wave 0 recon -> baseline measurement on disk
LOOP   fingerprint -> tasklist -> launch wave -> merge -> verify -> score ->
       persist  (repeat)
STALL  2 flat beats -> classify -> recover -> report pulse
FAIL   ticket -> ideas (>= 3 distinct) -> fingerprint best -> retry
ABORT  2 no-progress rounds / cap hit / same blocker twice / missing manual input
DONE   all DoD falsifiers green + honestscore >= T + evidence persisted +
       CONTINUE-HERE current
```

---

# APPENDIX I - SEEN-IN-THE-WILD: TWO ANTI-PATTERNS THIS SKILL REPLACED

## I.1 The vibes loop

Symptom: a run "iterated" 15 times, each iteration re-stating the plan with new
adjectives; no artifact changed between rounds 3 and 15.
Cause: no fingerprinting, no falsifiers, honestscore from memory instead of a run.
Fix: this skill - ledger + fingerprint + per-round evidence. The fingerprint
collisions in the ledger made the loop visible immediately.

## I.2 The optimistic scoreboard

Symptom: a status table showed 20/20 PASS while the bundle's own independent scan
showed 3/20. Stored flags had been trusted over fresh computation.
Fix: every aggregate is recomputed from artifacts by a different method than the
one that produced it; the independent recompute wins (this is also honestscore's
E-rule: a check must have been able to come out the other way).




---

# APPENDIX I - FIELD NOTES (real failures that shaped this skill)

## I.1 The stored-flags verdict

One real run declared "20/20 pass" as DoD while its own independent scanner measured
3/20 against the strict field spec. The stored flags and the strict detector were
measuring different line formats (a shipped minimum-guard line vs the canonical
14-field line) - both honest, but never reconciled, so the run "passed" against its
own stored flags while failing the spec. Fix: DoD criteria name the measurement
method AND the format it applies to; when two formats exist in the evidence, both
rates are reported, never just the flattering one.

An honest goal report shows the strict number next to the shipped-format number and
says which rule produced which. Anything else is the proxy-metric failure: ranking
your own evidence by the version of the metric that passes.
