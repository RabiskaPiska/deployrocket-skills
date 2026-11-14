---
name: spec-analyze
description: Cross-artifact consistency check for multi-artifact builds - the Spec Kit /speckit.analyze pattern. Read-only pass over spec<->plan<->tasks<->code that surfaces structural conflicts and gaps - a task with no requirement behind it, a plan step contradicting the spec, two artifacts assuming different interfaces, a requirement with no owner. Produces a read-only findings report (severity-graded, file-cited) BEFORE and DURING multi-wave builds; it does not edit, does not fix, does not replace review (correctness) or converge (completeness) - it is the consistency lens across artifacts. alwaysFire variant - fires automatically on wave-1 dispatch for any build with a written spec plus 2+ artifacts (plan, tasklist, briefs); situational variant - re-run on demand at any artifact boundary change.
alwaysFire: true
---

# spec-analyze - cross-artifact consistency check

You are running **spec-analyze**. A build has MORE THAN ONE artifact
describing it - a spec, a plan, a tasklist, spawn briefs, and eventually the
code. Every pair of artifacts is a place where two sessions' assumptions can
silently disagree. Your job: read ALL of them and report where they
contradict, where they leave gaps, and where they invent work nobody asked
for - BEFORE those disagreements are built into the product.

**The canonical defects this gate exists for (Spec Kit /speckit.analyze
taxonomy, fleet-corroborated):**

| Defect class | Shape |
|--------------|-------|
| Orphan task | a tasklist row no requirement demands (invented scope) |
| Orphan requirement | an R no task covers (the unassigned remainder) |
| Contradiction | plan step says X, spec says NOT-X |
| Duplication | two tasks/units claiming the same R (double scope - the subagents Rule 2 defect at the artifact level) |
| Drift | an artifact paraphrasing the spec and losing a constraint in the paraphrase |
| Boundary violation | a plan step reaching into the out-of-scope list |
| Unverifiable requirement | an R with no verify line surviving into the plan |

**What this skill is NOT:**

- NOT a code reviewer - it reads ARTIFACTS (spec, plan, tasks, briefs,
  design docs), not implementations. Correctness of code is the review
  gate; code-vs-spec completeness is converge. This skill is the map-level
  check between the PAPER artifacts.
- NOT a rewrite pass. Output = a findings report. Fixes happen in the
  owning artifact by its owner (one artifact, one writer).
- NOT a one-shot ritual. It is cheap enough to re-run at every artifact
  boundary (after planning, after each spec change, before each wave
  dispatch), and the situational trigger set says exactly when.

---

## 0) HARD RULES (survive compaction)

1. **Read-only.** The analyze pass never edits artifacts. It produces
   `<run-dir>/ANALYSIS.md` - findings, severities, evidence pointers. An
   analyze pass that edits is a review pass wearing the wrong badge.
2. **Every finding cites both artifacts** (file + line/section in each).
   "The spec and plan disagree" without quoting both sides is not a
   finding; it is a vibe.
3. **The spec is the authority on WHAT; the plan is the authority on HOW
   (within spec).** A how-conflict inside the plan is not a spec conflict -
   route it to ideas, not to the spec. A what-disagreement always resolves
   to the spec.
4. **Severity-graded:** Critical (blocks dispatch: contradiction or
   orphan-blocking requirement), Major (blocks the affected wave: drift,
   duplication, boundary reach), Minor (cosmetic inconsistency), Info.
5. **Coverage is computed, not remembered.** The R-to-task matrix is built
   mechanically (grep R-numbers across artifacts) - a matrix written from
   memory is a fiction with a table costume.
6. **Re-run on change.** Any artifact edit after the last analysis voids it
   (like the review-gate freeze rule): re-run before the next wave.
7. **No new requirements invented during analysis.** Spotting a missing
   requirement is a finding ("R-count looks wrong for this goal"), not a
   license to write R9 yourself - propose it, mark it proposed.
8. **Degraded path:** no spec file exists but a plan/tasklist does ->
   analyze plan-vs-tasklist consistency and report `spec: absent` honestly;
   do not refuse the pass, but the report is weaker and says so.

---

## 1) WHEN IT FIRES

Situational (multi-artifact builds) with these hard triggers:

| Trigger | Analyze scope |
|---------|---------------|
| Wave-1 dispatch for a build with SPEC.md + plan/tasklist | full pass before spawning (the plan is about to be executed - this is the last cheap moment to catch conflicts) |
| SPEC.md edited (changelog line added) | re-run: spec-vs-plan + spec-vs-tasks |
| Tasklist rewritten / re-planned after a wave | re-run plan-vs-tasks |
| New artifact enters (briefs, design doc, runbook) | pairwise vs the spec |
| Pre-merge (crew builds) | cross-fragment consistency (names, interfaces, boundaries) |
| "Why does the plan do X?" confusion mid-build | targeted single-pair pass |

Skip conditions (honest skips): `skipped: no artifacts to cross-check`
(single-artifact task - converge covers it), `skipped: no spec and nothing
to pair` (free-form micro-build).

---

## 2) THE PROCEDURE (step by step)

**S1. Load the artifact set.** Enumerate what exists: SPEC.md, plan file,
tasklist, briefs, design docs, brain references. Record the version (hash
or date) of each - the analysis is versioned against exactly these.

**S2. Build the coverage matrix.** For every R in the spec: which plan
steps and tasklist rows cite it? For every tasklist row: which R cites it?
The matrix is mechanical (grep R-numbers); the ANALYSIS is reading the
rows:

```
matrix (illustrative):
       R1  R2  R3  R4  R5  R6
T1     x
T2     x   x
T3             x
T4                     x
T5                             <- orphan: no R
R6                     <- orphan: no task
```

**S3. Consistency pass per pair.** For each artifact pair (spec-plan,
plan-tasks, spec-tasks, briefs-spec):
- Contradictions: literal conflicts (spec: no XLSX; plan step 4: add XLSX).
- Drift: paraphrases that weakened constraints (spec: "stream files over
  100MB"; brief: "handle large files").
- Duplicates: two units owning the same R with no handoff named.
- Boundary reach: any step touching the out-of-scope list.

**S4. Interface check across fragments (multi-seat builds).** Shared names
(config keys, agent ids, function signatures, paths) spelled consistently
in every artifact that mentions them; assumptions pinned in one place and
referenced elsewhere (the subagents shared-assumptions rule, checked
mechanically).

**S5. Grade + write ANALYSIS.md.** Fixed structure (Section 5). The report
ends with a dispatch verdict: `CLEARED | BLOCKED (n Critical) |
CLEARED-WITH-FIXES (majors scheduled before wave N)`.

**S6. One-line announce.** `analyze: 1 critical, 2 major -> BLOCKED
(S2 re-plan)` or `CLEARED` - into the gates line and the runlog.

---

## 3) SEVERITY LADDER (graded precisely)

| Severity | Definition | Effect |
|----------|-----------|--------|
| Critical | two artifacts command contradictory work, or a blocking requirement has no owner, or a boundary violation reaches the out-of-scope list | BLOCK dispatch until resolved (clarify escalation or plan fix) |
| Major | drift that loses a constraint; duplicated scope; orphan task inventing scope | fix before the affected wave; briefs updated |
| Minor | naming inconsistency between artifacts; stale cross-reference | fix at next natural edit, logged |
| Info | wording improvement, formatting | optional, never blocking |

Two examples calibrated:

- Spec R3 says exporter must stream files over 100MB; the plan builds an
  in-memory path "for simplicity". -> **Critical** (the plan commands
  work the spec forbids; dispatching builds throwaway work).
- Tasklist has T7 "polish dashboard styling" while the spec's UI
  requirement is R7 (admin gate). T7 has no R. -> **Major** (invented
  scope - either delete T7 or write the R that justifies it, owner call).

---

## 4) THE COVERAGE MATRIX - mechanics

Build it mechanically, judge it manually:

```
grep -o "R[0-9]\+" <plan> <tasks> <briefs/*> | sort | uniq -c
# per R: which artifacts cite it; per artifact: which R's it cites
```

Reads that matter:

- An R cited by NO task = orphan requirement (converge would catch it
  later; analyze catches it now, before the waves run - cheaper).
- A task citing NO R = orphan task (invented scope - justify or delete).
- An R cited by MULTIPLE tasks = either duplication (defect) or a named
  handoff (fine - the boundary must name the handoff, subagents Rule 2
  applied to artifacts).
- The matrix is a PICTURE of the plan, not the verdict; the verdict is the
  graded analysis of what the matrix shows.

---

## 5) ANALYSIS.MD - fixed structure

```
# ANALYSIS - <task> (<date>, spec v<hash>)
artifacts: SPEC.md v<h> | plan.md v<h> | tasklist.md v<h> | briefs/*.md

## Coverage matrix
<the R x artifact matrix or its summary>

## Findings
### F1 | Critical | spec-plan contradiction
spec: SPEC.md R3 - "WHEN file >100MB THEN stream to disk"
plan: plan.md step 4 - "load dump, process, write once"
both quoted. resolution options: (a) fix plan, (b) amend spec
-> routed: <owner/plan-fix>, deadline: before wave 2

### Dispatch verdict
CLEARED | BLOCKED (F1) | CLEARED-WITH-FIXES (F2 before wave 2)
```

The both-quoted rule is what makes findings decidable: a reader (or the
next session) can re-adjudicate every finding from the report alone,
without this session's context. That is the same evidence standard the
review gate and converge enforce.

---

## 5) VERIFICATION COMMANDS

```
# V1. The analysis exists and names versions
grep -c "v<" <run-dir>/ANALYSIS.md   # or explicit version lines - expect >= artifact count

# V2. Every finding cites both artifacts
grep -c "^### F[0-9]" <run-dir>/ANALYSIS.md        # count
grep -c "spec:" <run-dir>/ANALYSIS.md              # both-anchor count >= findings

# V3. Coverage matrix actually computed (not asserted)
grep -c "R[0-9]" <run-dir>/ANALYSIS.md >= R-count in SPEC.md

# V4. Void-on-edit check
#    if any artifact mtime > analysis mtime -> analysis is stale -> re-run
```

V4 is the drift guard: an analysis is a statement about specific versions;
edits after it are new facts, not amendments.

---

## 6) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Analysis finds nothing on a build that later diverges | matrix built from memory, not grep | V3-style mechanical build; the grep IS the matrix |
| Contradiction found but dispatch happened anyway | verdict not enforced | BLOCKED means no spawns; the once-per-rollout bypass test applies (review V3 analog) |
| All findings are Minor (rubber-analyze) | severity ladder misread - contradictions graded down | re-grade with the definition: contradictory work commands = Critical regardless of size |
| Analyze rewritten the plan while "checking" | read-only violated | finding goes to the report; plan edits belong to the planner seat |
| Drift found between briefs and spec AFTER wave 1 | briefs written before analysis | order: analyze -> briefs -> dispatch; briefs citing R-numbers verbatim |
| Duplication found mid-run (two seats, one R) | no handoff named | subagents Rule 2: merge or rewrite boundary; log the merge |

---

## 7) ANTI-PATTERNS

- THE RUBBER STAMP: "artifacts consistent" with no matrix. The analysis
  without the matrix is a mood, not a check.
- THE PLAN REWRITE: analysis drifting into redesign. Findings + routing;
  the owning artifact's owner fixes.
- THE ONE-TIME PASS: analyzing once at intake and never again despite
  artifact edits. Void-on-edit rule exists for exactly this.
- THE SILENT DRIFT ACCEPTANCE: "the brief basically says the same thing."
  Paraphrase is where constraints die; quote, compare, cite.
- ANALYSIS AS GATEKEEPER-THEATER: running it on a two-file build where the
  spec IS the plan (single artifact - nothing to cross). Skip honestly:
  `analyze: skipped: single artifact (converge covers it)`.

---

## 8) WORKED EXAMPLES

### 8.1 The dashboard build (multi-artifact, real shape)

Spec R1-R6; plan has 6 steps; tasklist has T1-T6 across 3 seats.
Coverage matrix run: R5 (export) cited by no task; T5 ("polish styling")
cites no R. Plan step 4 says "load dump into memory"; spec R3 says stream
over 100MB. Findings: 1 Critical (R3 contradiction - dispatch would build
throwaway code), 1 Major (T5 invented), 1 Major (R6 unowned). Verdict:
BLOCKED. Re-plan (step 4 rewritten to stream; T5 deleted pending owner;
R6 assigned to seat 2's T3). Re-analysis on the new versions: CLEARED.
Wave 1 dispatches. The uncaught version of this exact shape - "plan
quietly contradicts spec on the data path" - is the shipped-then-rewritten
build class this skill prevents.

### 8.2 Brief drift (the paraphrase killer)

Brief for seat 3 says "persist settings between runs". Spec R4 says
"settings persist across restarts via settings.json, loaded before any
export". The brief's paraphrase dropped "loaded before any export" - a
constraint that determines initialization order. The pair-pass
(brief-spec) catches it; the brief is rewritten citing R4 verbatim before
dispatch. Cost: one read. The uncaught version is an init-order bug that
review would eventually find - after two seats built against different
assumptions.

### 8.3 Honest skip

Task: fix one flaky test. Artifacts: the spec (3 lines) and nothing else -
no plan, no tasklist. `analyze: skipped: single artifact (converge covers)`
- correct. Running the pair-pass on one artifact is theater.

---

## 9) GATES-LINE CONTRACT

Slot in the per-task extension:

```
analyze: CLEARED (3 artifacts, 0 findings) | analyze: BLOCKED (1 critical: <one line>)
analyze: CLEARED-WITH-FIXES (2 majors pre-wave-2) | analyze: skipped: single artifact
```

Feeds `dim-completeness` (coverage matrix evidence) and
`dim-correctness` (contradiction findings = correctness evidence at the
artifact level, complementing the review gate's code-level pass).

When reporting skipped, the honest whys: `skipped: single artifact`,
`skipped: no spec (converge self-serve)` (still runs plan-vs-tasks if both
exist), `skipped: trivial ask`.

---

## 10) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| spec-clarify-gate | supplies SPEC.md - the authority artifact this skill reads; no spec, degraded pass |
| subagents | runs BEFORE wave-1 briefs; briefs must cite R-numbers verbatim (drift check) |
| spec-converge | the end-of-build twin: analyze checks artifact-vs-artifact BEFORE build, converge checks code-vs-spec AFTER; same R-universe |
| auto-review-gate | review reads the merged artifact; analyze reads the artifacts about the artifact - different layers, no overlap |
| auto-review-gate (join reviews) | the interface/name consistency half of analyze continues at every join |
| ideas | Major+ findings become typed tickets |
| session-monitor-auto | analysis is a step like any other; stall rules apply |

---

## 11) OPEN ITEMS (honest limits)

- The coverage matrix is name-based (R-number citations); a task satisfying
  an R without citing it reads as an orphan - the matrix's false-positive
  direction. Judge orphans before deleting: the T5-polish case may be a
  legitimate non-functional task whose R is missing (write the R, do not
  delete the work).
- No tooling; the matrix is grep + judgment. A matrix generator is a
  natural companion script - noted, not built.
- Severity grading is judgment; the both-quoted rule is the control
  (evidence-gated grading).

---

## APPENDIX A - THE ONE-CARD OPERATING SUMMARY

```
ENUMERATE  list the artifacts + version each (hash/date) - analysis is
           versioned to exactly these
MATRIX     grep R-numbers across plan/tasks/briefs -> coverage picture
CROSS      per pair (spec-plan, plan-tasks, spec-briefs, briefs-spec,
           spec-out-of-scope): contradictions | drift | duplicates
GRADE      Critical (contradiction / blocking orphan / boundary) |
           Major (drift, duplicate, orphan) | Minor | Info - both quotes
WRITE      ANALYSIS.md: versions, matrix, findings, dispatch verdict
VERDICT    CLEARED | BLOCKED (no spawns until fixed) | CLEARED-WITH-FIXES
RE-RUN     on any artifact edit (targeted pairs) - void-on-edit
NEVER      memory-built matrix | rubber "consistent" | rewriting artifacts |
           accepting paraphrase | one-time-pass on a living build
```

---

## APPENDIX B - PAIR-PASS CHECKLIST (the six pairs, in order)

1. spec vs goal paragraph (spec-internal contradiction - drill #10)
2. spec vs plan (the what-vs-how contradictions)
3. plan vs tasklist (every step has tasks, every task a step)
4. spec vs briefs (drift - the paraphrase killer)
5. briefs vs tasklist (seat boundaries + shared assumptions pinned)
6. artifacts vs out-of-scope list (boundary reach - even a stub counts)

Each pair: read BOTH fully, quote conflicts from both sides, grade, move
on. Full walk at intake and before the final wave; targeted pairs after
trigger events (Appendix G).

---

## APPENDIX C - GRADING DRILLS (calibrate severity by consequence)

Grade by what WORK dies or doubles, never by embarrassment or size:

1. Plan streams vs spec in-memory (or vice versa) on the data path ->
   Critical (throwaway work).
2. Brief drops a load-bearing number ("loaded before any export") ->
   Major (constraint drift).
3. Two seats, one R, no named handoff -> Major (Rule 2 at artifact level).
4. Plan step creates a scheduler stub while the out-of-scope list says no
   scheduling -> Critical (the creep lives even as a stub).
5. Spec-internal conflict (R1 idempotent vs R5 unique filenames) ->
   Critical, routed to clarify - never "fixed" silently in the plan.
6. Task citing no R ("polish styling") -> Major (justify or delete).
7. Stale analysis after spec edit -> void, re-run targeted pairs.

---

## APPENDIX D - WORKED EXAMPLE: THE DUPLICATE HANDOFF DONE RIGHT

Two seats both cite R4 (settings persistence). Naive read: duplication
defect. The pass demands the boundary: seat 2's T3 = "settings WRITE path
(settings.json schema, atomic writes)"; seat 3's T4 = "settings READ path
(load at startup, feed exporter config)". The boundary names the handoff
explicitly (seat 3 consumes the schema seat 2 writes; schema file pinned
in shared assumptions). Verdict: CLEARED - a double citation with a named
handoff is legitimate (subagents Rule 2 permits exactly this).

The failure version (same matrix, no named handoff): both seats implement
their own settings.json parser. Two parsers, two failure modes, a merge
conflict git cannot resolve (semantic). The matrix flagged it; the
boundary question decided it - which is why the analysis runs BEFORE
dispatch, when the boundary is still a one-line fix.

---

## APPENDIX E - DEGRADED PASSES (no spec / partial artifacts)

| Artifact set | Legal pass |
|--------------|-----------|
| plan + tasks, no spec | plan-vs-tasks pair only; report `spec: absent`; converge runs self-serve at close |
| spec only (no plan/tasks) | nothing to cross - `analyze: skipped: single artifact` |
| spec + briefs (no plan) | spec-vs-briefs pair; briefs must cite R-numbers verbatim |
| everything exists | full six-pair walk |

Degraded passes stay read-only, both-quoted, and versioned. An analysis
against unnamed versions is void by construction.

---

## APPENDIX F - THE FULL WORKED PASS (annotated)

Scenario: "add export + settings", 3 seats planned.

```
S1 artifacts: SPEC.md v4b2 | plan.md v7 | tasklist.md v6 | briefs x3

S2 matrix (grep-built):
   R1 x T1 | R2 x T1,T2 (handoff named) | R3 x T3 | R4 x T3,T4 (handoff?)
   R5 cited by NO task | T5 cites NO R

S3 pairs:
   spec-plan  : F1 Critical - plan step 4 "load dump into memory" vs
                spec R3 "WHEN file >100MB THEN stream to disk"
   spec-briefs: F2 Major - seat-3 brief drops "loaded before export"
   boundary   : clean

S4 grade + verdict: BLOCKED (F1). routed: plan rewrite (streaming), brief
   fix (cite R4 verbatim), R5 assigned, T5 deleted pending owner.

re-run on new versions: CLEARED. wave 1 dispatches.
```

The uncaught version of F1 is the shipped-then-rewritten build: two seats
building against contradictory data paths, caught by converge only after
the code exists.

---

## APPENDIX G - ANALYZE ON LIVING BUILDS (re-run triggers, exactly)

Re-run (targeted to the changed artifact's pairs) when ANY of:

- The spec gained or changed an R (changelog line).
- A wave's merge rewrote a shared interface or shared-assumptions file.
- A brief was rewritten.
- A divergence was routed (the resolution edits an artifact).
- A join review reworked a boundary.

Full passes: at intake and before the final wave. Targeted: between. The
stale-re-run metric catches skipping.

---

## APPENDIX H - WHY PRE-WAVE AND NOT "WHEN THERE IS TIME"

Economics, measured class: a finding fixed at analysis time costs a
one-line artifact edit (minutes). The same contradiction found at wave 2
costs the throwaway build + the fix dispatch + the re-verify. The analysis
is the cheapest gate in the chain and the only one that runs when nothing
has been built yet - which is why it is mandatory before wave 1 and
optional (targeted) elsewhere. Skipping the cheap gate to save time is the
most expensive line item in the measured campaign log.

---

## APPENDIX I - METRICS (one line per analysis)

```
analyze | <date> | spec v<h> | artifacts <count> | findings c<C> m<M> i<I>
| orphans R<list> T<list> | verdict <CLEARED|BLOCKED|W-FIXES>
| pre-wave <yes|no> | stale-re-runs <count>
```

Signals: findings rate (0 findings across 3 analyses on a multi-artifact
build = check for rubber-stamping); pre-wave rate (late analyses = the
order rule is slipping); stale re-runs (churn without re-analysis).

---

## APPENDIX J - FAQ (short, factual)

**Q: The task "obviously" satisfies an R without citing it.**
Then cite it - the citation is one line and the matrix exists to be
mechanical. Uncited work reads as orphan; judgment on orphans happens
BEFORE deletion (write the missing R instead, if the work is real).

**Q: Contradiction but the plan's way is obviously better?**
Then the SPEC is wrong: route a spec amendment through the changelog
(owner or the authorized decision path), and record the analysis finding
as resolved-by-spec-edit. The analysis never edits the spec, and the plan
never wins silently.

**Q: Two analyses disagree?**
They analyzed different versions (V4 stale rule). Version-stamp every
analysis; disagreement across versions is not a conflict - re-run on
current.

**Q: Is analyze needed on a solo build (no crew)?**
If a plan and tasklist exist, yes - the pairs are cheaper than the rework
even solo. If the spec IS the plan (single artifact), skip honestly.

**Q: What enforces the BLOCK verdict?**
The dispatch gate: briefs do not go out while BLOCKED (subagents wave
discipline). The once-per-rollout bypass test (deliberately leave a
Critical open; the pipeline must refuse) proves the enforcement exists -
the same self-attack the review gate's V3 runs.

---

## APPENDIX K - THE VOID-ON-EDIT GUARD, PROVEN (install test)

Like every gate in this fleet, the analysis gate does not exist until
proven to fail on its own defect class. The proof procedure (run once per
rollout):

1. Run the analysis; record CLEARED and the artifact mtimes.
2. Edit ONE artifact (add a contradictory plan step).
3. Run the next dispatch attempt - the gate must refuse or flag
   `analysis stale (mtime newer than analysis)`.
4. If dispatch proceeded silently, the void-on-edit guard does not exist;
   fix the order enforcement before trusting any CLEARED verdict.

This is the "a guard does not exist until proven to fail on the defect it
was written for" rule, applied to the analysis layer - the same
once-per-rollout self-attack as review V3 and converge's bypass test.

Dim feeding note for the report: a pre-wave CLEARED analysis with the
coverage matrix pasted is direct `dim-completeness` evidence at the
artifact layer; cite it alongside the converge table at close.

---

## APPENDIX L - SECOND WORKED PASS: THE SOLO BUILD (no crew)

Solo builds still have artifact pairs - the spec and the plan you wrote
yourself, which is exactly where self-deception lives:

```
S1 artifacts: SPEC.md (R1-R4) | plan.md (the approach sketch you wrote
   at intake - same session, same brain, zero fresh eyes)
S2 matrix: R4 (verify: manual steps) has no verify command; R2's verify
   references a script that does not exist yet
S3 findings: F1 Major - two R's with broken/unwritten verify lines (the
   build would have been unassessable at converge)
S4 fix: verify lines written (one test, one manual procedure), spec
   changelog updated, re-analysis CLEARED
```

Solo drift is the most dangerous kind - there is no second seat to notice.
The analysis pass is the only fresh-eyes moment before the code exists.

---

## APPENDIX M - EXTENDED TROUBLESHOOTING (classes beyond Section 6)

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Analysis says CLEARED, wave 1 briefs still drift | briefs written before the analysis landed | order: analyze -> briefs -> dispatch (the chain order rule) |
| The matrix is huge and unreadable (30+ R's) | flat R-list on a big build | group R's by requirement area; the matrix works per-group with a roll-up |
| Findings counted but nothing routed | findings with no `routed:` line | every finding names its resolution path (plan-fix / spec-edit / owner / delete); unrouted findings are the stall |
| Analysis buried in chat, file never written | "the report" stayed prose | ANALYSIS.md is the deliverable; chat is the derivative |
| Repeated identical findings across re-runs | the routed fix never landed | the re-run diffs against the previous ANALYSIS.md; identical findings = the fix dispatch failed (footprint check) |
| Severity inflation (everything Critical) | consequence grading skipped | re-grade with the drill (Appendix C): what work dies or doubles? |

---

## APPENDIX N - THE COVERAGE MATRIX, FULL WORKED OUTPUT

For SPEC.md R1-R6, plan.md, tasklist.md (T1-T6), briefs (3 seats) - the
actual grep and its reading:

```
$ grep -oE "R[0-9]+" plan.md tasklist.md briefs/*.md | sort | uniq -c
   6 R1 plan.md
   4 R1 tasklist.md
   2 R2 plan.md
   2 R2 briefs/seat1.md
   1 R3 plan.md
   3 R3 briefs/seat2.md
   2 R4 tasklist.md
   2 R4 briefs/seat3.md
   1 R6 plan.md
   1 T5 tasklist.md          <- a TASK id in the R-grep: noise, ignore
   0 R5 anywhere             <- ORPHAN requirement
   (no R-citation lines for tasklist T5 -> candidate orphan task)
```

Reading, step by step:

1. R5 (export UI) cited nowhere -> orphan requirement -> assign an owner
   BEFORE dispatch (converge would catch it at close; analyze catches it
   at intake - the difference is one edit vs one built-wrong wave).
2. T5 ("polish styling") has no R-citation -> orphan task -> justify or
   delete (write the R if the work is real).
3. R4 cited by two seats -> demand the named handoff (Appendix D) or
   merge.
4. R6 cited once, by T3 - single owner, fine.
5. The lone task-id hit is grep noise (T5 matched the R-pattern) - a
   reminder that the matrix is a picture, not a verdict.

The full matrix on a real build takes minutes; the judgment layer (which
orphans are real) is the analysis.

---

## APPENDIX O - ARTIFACT VERSION STAMPS (the exact format)

Every analysis opens with the version block; every downstream record
(re-runs, verdicts, converge) cites it:

```
artifacts:
  SPEC.md        v<git-hash or mtime-hash>  (<size> lines, <date>)
  plan.md        v<...>
  tasklist.md    v<...>
  briefs/seat1.md v<...>
```

Why hashes and not "current": analyses get cited later ("the CLEARED
analysis said...") - a citation without a version is unfalsifiable. The
hash costs one `git rev-parse` per artifact (or a `Get-FileHash` on
non-repo files). The stale rule becomes mechanical: compare the stamp, not
the memory.

---

## APPENDIX P - THE ORDER RULE IN THE CHAIN (where analyze sits)

The full chain, with the analysis's position marked:

```
intake -> spec-clarify-gate (SPEC.md) -> SPEC-ANALYZE (pre-wave) ->
  planner (plan.md, tasklist) -> re-analyze (targeted, plan changed) ->
  briefs (R-verbatim) -> dispatch check (BLOCK enforcement) ->
  wave 1..N (join reviews at merges) -> merge ->
  converge (code vs spec) -> review (correctness, fresh) -> honestscore
  -> done-claim
```

Two placement rules worth restating:

1. The FIRST analysis runs before the planner writes briefs - briefs that
   cite a pre-analysis plan inherit its contradictions into every seat.
2. The LAST analysis (targeted) runs before the final wave, so converge
   assesses a build whose artifacts were consistent at the last edit.

Between those two anchors, re-runs are trigger-driven (Appendix G).

---

## APPENDIX Q - WRITING FINDINGS THAT DECIDE (the resolution discipline)

Every Major+ finding ends with a routed resolution line; the four legal
routings:

```
-> routed: plan-fix (the plan contradicts the spec; planner edits plan)
-> routed: spec-amend (the spec was wrong; changelog entry; owner path
   if the change is scope)
-> routed: owner (a real product decision - one question, recommended
   option; the clarify escalation path)
-> routed: delete (orphan task; no R justifies it and none will)
```

A finding without a routing line is a complaint, not a finding - it will
be re-found at every re-run (the identical-findings signal in Appendix M).
The routing deadline is named in the finding (`before wave 2`), because a
Major with no deadline floats forever while the build proceeds on the
contradiction.

Anti-pattern observed in the wild: findings routed to "TODO". A routed
finding names a WHO and a WHEN; TODO is neither.

---

## APPENDIX R - INTERACTION WITH CONVERGE (division of labor, precisely)

The two passes share the R-universe but never the same evidence:

| Aspect | spec-analyze (pre/during) | spec-converge (post) |
|--------|---------------------------|----------------------|
| Inputs | spec, plan, tasklist, briefs | the built artifact + SPEC.md |
| Question | do the ARTIFACTS agree? | does the CODE match the spec? |
| Output | findings + routed resolutions | per-R verdicts + e2e + appended tasks |
| Blocks | dispatch | the done-claim |

The overlap edge case: an analysis finding routed to plan-fix that was
never applied becomes a converge finding at close (the code now embodies
the contradiction). The two gates triangulate the same defect from
opposite ends - converge's table row for the affected R cites the
analysis finding that predicted it, which is the cleanest possible
completeness evidence.

What this division prevents: double-judging the same defect (analyze owns
artifact-level, review owns code-level, converge owns build-vs-spec), and
the gap where each gate assumed another checked it.

---

## APPENDIX S - THE ANALYZE SEAT DEFINITION (agent file, drop-in)

```yaml
name: spec-analyzer
mode: subagent
tools: [read, grep, glob, bash]
max-steps: 16
prompt: |
  You are a read-only consistency analyst for a multi-artifact build.
  You receive: SPEC.md path, plan/tasklist/brief paths, the previous
  ANALYSIS.md (if any).

  Procedure:
  1. Read every artifact FULLY. Build the R-citation matrix with grep.
  2. Walk the six pairs (Appendix B). Quote both sides of every conflict.
  3. Grade by consequence (drill). Route every Major+ finding (who+when).
  4. Write <run-dir>/ANALYSIS.md in the fixed structure.
  5. Verdict: CLEARED | BLOCKED | CLEARED-WITH-FIXES.

  OUT: you never edit artifacts. You never spawn subagents. You do not
  propose designs - findings + routings only. If an artifact is missing,
  record it and continue with the degraded pass (Appendix E).

  Final: fenced JSON {"status","verdict","findings":[],"orphans":[]}.
```

Pinning the analyzer as an agent definition makes the read-only guarantee
structural (the platform denies write tools) and gives every analysis the
same procedure without re-deriving the mandate per spawn.

---

## APPENDIX T - COUNTING RULES (what the numbers mean)

The metrics line uses these exact definitions (so runs are comparable):

- `findings c/m/i`: Critical/Major/Info counts at THIS analysis.
- `orphans`: requirements with zero citations (R-list) and tasks with
  zero citations (T-list) - listed by id, judged before action.
- `pre-wave yes|no`: whether the analysis completed before any brief was
  dispatched. Late analyses count toward the pre-wave-no signal.
- `stale-re-runs`: dispatch attempts that had to be preceded by an
  unplanned re-analysis (churn happened without a trigger).

A build's analysis history should show: pre-wave yes, findings strictly
decreasing across re-runs (each finding resolved by its routing), stale
re-runs trending to zero. Any other shape is a process smell - the
economics appendix (H) says why it is also the expensive shape.

---

## APPENDIX U - ARTIFACT VERSION STAMPS (the exact format)

Every analysis opens with a version block; every downstream record cites it:

```
artifacts:
  SPEC.md         v<git-hash or mtime-hash>  (<size> lines, <date>)
  plan.md         v<...>
  tasklist.md     v<...>
  briefs/seat1.md v<...>
```

Why hashes and not "current": analyses get cited later (by converge, by
review, by the next session). A citation without a version is
unfalsifiable. The stale rule becomes arithmetic: if any artifact's
current hash differs from the stamp, the analysis is void for the pairs
touching that artifact (targeted void, not full void) - re-run those
pairs before dispatch.

---

## APPENDIX V - GATES-LINE EXTENSION DETAILS

The per-task extension slot for this skill:

```
analyze: CLEARED (spec v4b2, 3 artifacts, 0 findings, pre-wave)
analyze: BLOCKED (1 critical: plan step 4 contradicts R3)
analyze: CLEARED-WITH-FIXES (2 major, routed plan-fix, before wave 2)
analyze: skipped: single artifact | skipped: no spec (plan-vs-tasks only)
```

Values never contain the pipe character. A skipped analysis on a
multi-artifact build is a gate bypass and gets counted by the audit.
Report rule: the completion report cites the ANALYSIS.md path and the
verdict - a CLEARED claim without the written analysis file is a lying
completion report by fleet rule.

---

## APPENDIX W - FINAL SELF-CHECK (before any CLEARED verdict)

1. Did I read every artifact FULLY (not summaries)?
2. Was the coverage matrix built by grep (not recalled)?
3. Does every Major+ finding quote BOTH sides and carry a routing (who+when)?
4. Are the artifact version stamps in the report header?
5. Is ANALYSIS.md written to the run dir (not only in chat)?
6. If any artifact changed since intake - did the targeted re-run cover its pairs?

Any "no" = the pass is not done. The analysis gate is the cheapest in the
chain precisely because its inputs are text; there is no excuse for an
expensive version of it.
