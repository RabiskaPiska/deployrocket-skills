---
name: spec-converge
description: End-of-build completeness gate - after implementation, re-assess the built code against SPEC.md/plan/tasks, APPEND every missed requirement as a new task (append-only tasklist), and loop build-vs-spec until the spec declares Converged. The missing twin of honestscore - honestscore judges QUALITY of what was built, spec-converge judges COMPLETENESS against what was ASKED. The Spec Kit /speckit.converge pattern (assess -> append missed work as tasks -> re-run) adapted to the fleet. alwaysFire - fires at the end of every build before the done-claim; the done-claim is illegal while convergence is UNCONVERGED with open spec items.
alwaysFire: true
---

# spec-converge - close the gap between what was built and what was asked

You are running **spec-converge**. The build phase ended. Code exists. Tests
maybe pass. Before ANY done-claim, you re-assess the code against SPEC.md -
the frozen requirements file from spec-clarify-gate - and you APPEND every
piece of missed work as a new task, then loop until the spec is covered or
the divergence is explicitly justified.

**Why this exists (the measured failure class):** honestscore judges the
quality of what was built; nothing in the fleet judges whether everything
ASKED was built. The two failures look identical from inside ("tests pass,
looks done") and diverge the moment the user opens the deliverable and finds
the second half of the request missing. Spec Kit's converge phase exists for
exactly this; the campaign's morning reports show the same class - work
"done" that never covered requirements 5 through 9 because plan-iteration
stopped while momentum was high.

**What this skill is NOT:**

- NOT honestscore. Quality vs completeness: honestscore asks "is what
  exists good"; spec-converge asks "is what was asked present". They run in
  that order (a converged-but-broken build fails honestscore; a beautiful
  partial build fails converge).
- NOT a re-planning skill. It does not redesign; it APPENDS missed work as
  tasks against the existing plan structure.
- NOT optional at the end of builds. It is the twin gate: no converge, no
  done-claim.

---

## 0) HARD RULES (survive compaction)

1. **The spec is the checklist; the code is the suspect.** Assessment walks
   R1..Rn in SPEC.md and marks each `implemented | partial | missing |
   diverged` with file:line evidence. Never the reverse direction ("look at
   the code and see what it satisfies" - that is how requirements quietly
   disappear).
2. **Append-only task growth.** Missed work becomes NEW tasks appended to
   the tasklist (never edits to completed rows, never silent scope
   shrinking). The tasklist is append-only during convergence, exactly like
   the beats ledger.
3. **Loop until Converged - with a cap.** Assess -> append -> build ->
   re-assess. The loop runs until every R is `implemented` or explicitly
   re-scoped in the spec's changelog. Hard cap: 3 convergence rounds; open
   items at the cap are reported honestly (same arithmetic as the review
   gate's 2-round rule).
4. **Divergence is a finding, not a feature.** If the code does something
   DIFFERENT from the spec and the code is right, the SPEC gets a changelog
   line and the R is marked `amended` - the spec moves by writing, never by
   drifting.
5. **The end-to-end check runs.** SPEC.md's end-to-end check (the single
   command or interaction that proves the whole) must actually run and
   pass. A green unit suite with a failing e2e check is UNCONVERGED.
6. **No new scope invented at convergence.** Finding a missing feature is
   not license to build extra features. The R-list is closed; discoveries
   about NEEDED work go through the R-append path (with a changelog line),
   not through silent implementation.
7. **The convergence verdict is written.** `<run-dir>/CONVERGENCE.md`:
   status (CONVERGED / UNCONVERGED n open / CONVERGED-WITH-EXCEPTIONS),
   per-R table, appended tasks, exception list. Downstream (gates line,
   morning report) reads the file.
8. **Out-of-scope violations are defects.** If the build touched something
   the spec's out-of-scope list named, that is a finding - either revert it
   or move it into scope through a written spec change. Silent boundary
   crossing is how "small favor" creep eats builds.

---

## 1) WHEN IT FIRES

Always-fire at BUILD END - the moment the last build wave's merge is
claimed complete:

| Trigger | Action |
|---------|--------|
| Build task merge completes (single or crew) | full convergence pass |
| subagents orchestrator finishes merge | converge BEFORE the verify wave's done-claim |
| goal-skill iteration claims a build milestone done | converge against the iteration's spec slice |
| Deploy followed a build | converge on deploy evidence too (R's that say "deployed" get probe evidence) |
| Research / read-only turn | skip (`converge: skipped: not a build`) |

---

## 2) THE ASSESSMENT TABLE - the core artifact

For each requirement in SPEC.md, exactly one verdict:

```
| R# | requirement (short) | verdict | evidence | gap |
|----|--------------------:|---------|----------|-----|
| R1 | render table from json | implemented | src/table.ts:41, test passes | - |
| R2 | cached fallback on API fail | partial | fallback exists; no "last updated" banner | banner + test |
| R3 | dim stale prices | missing | no date logic found anywhere | implement + verify |
| R4 | deploy probe | diverged | deploy script checks port only, spec says health+content | align or amend spec |
```

Verdict definitions (exact):

- `implemented` - the requirement's own verify line was run THIS session and
  passed, or the manual steps were executed and observed.
- `partial` - some of the requirement exists; the gap is named in one line.
- `missing` - nothing in the codebase satisfies the R (grep'd, not guessed).
- `diverged` - code satisfies a DIFFERENT requirement than written. Requires
  a judgment: fix the code or amend the spec (Rule 0.4) - both paths are
  written decisions.
- `removed` - the R was removed from the spec (changelog); still listed so
  coverage tables stay aligned across sessions.

The table lives in CONVERGENCE.md. Per-R evidence is mandatory: "looks
implemented" is not a verdict, it is a guess.

---

## 3) THE PROCEDURE (step by step)

**S1. Load the spec.** Read SPEC.md (requirements + changelog + out-of-scope
+ end-to-end check). If no SPEC.md exists, reconstruct the minimum spec
self-serve (spec-clarify-gate self-serve path) and mark the run
`converge: self-serve-spec` - a converge against no written spec is a
converge against memory, which is the failure class this skill prevents.

**S2. Build the evidence table.** For each R: locate the implementation
(grep the R's key phrase, trace the files the plan touched), run its verify
command, record verdict + evidence. Batch R-verify commands where possible
(one test run covering many R's is fine - cite which R's each run covers).

**S3. The end-to-end check.** Run the spec's end-to-end check verbatim. Pass
-> note the output. Fail -> the build is UNCONVERGED regardless of per-R
status (the e2e check outranks per-R greens - the join can be broken while
every fragment passes).

**S4. Append the missed work.** Every `partial`, `missing`, and `diverged`
row becomes an appended task in the tasklist (ID continues the sequence,
`depends-on` empty, wave = next). The append notes the R-number it closes.
No owner change: the orchestrator assigns; existing seats may absorb, or a
new dispatch runs them (subagents failure policy applies).

**S5. Fix wave.** Execute the appended tasks. The convergence loop's fix
dispatches follow the same rules as review fixes: ONE dispatch carrying the
complete list; resume the original implementer for rounds 1-2; fresh worker
from round 3 (or at the orchestrator's discretion when the original is
gone).

**S6. Re-assess.** Fresh pass over the FULL R-table (not just the new
items - fixes regress neighbors; the re-run is cheap because verify lines
exist). Update CONVERGENCE.md.

**S7. Verdict.**
- All R's `implemented` + e2e passes -> `CONVERGED`. Done-claim may fire
  (then honestscore judges quality).
- Items explicitly re-scoped in the spec changelog -> `CONVERGED-WITH-
  EXCEPTIONS` + the exception list in the report (honest, not hidden).
- Open items at the 3-round cap -> `UNCONVERGED (N open)` -> the done-claim
  is BLOCKED; report what is open and why, exactly like the review gate.

---

## 3) THE CONVERGENCE LOOP AS ARITHMETIC

State machine per build:

```
START -> ASSESS -> [all R implemented AND e2e pass] -> CONVERGED -> done-claim
              |-> open items -> APPEND tasks -> BUILD -> ASSESS (round+1)
              |-> round > 3 -> UNCONVERGED(cap) -> report open items, stop
```

Loop rules:

- Each round must show NET-NEW convergence (R's moving to implemented).
  Two consecutive rounds with zero net-new = the loop is stuck; stop and
  report (same abort arithmetic as the goal skill's iteration rules).
- The cap is 3 rounds of ASSESSMENT, not 3 fix dispatches; a round may
  carry many fixes. Do not re-assess after every single fix (that is the
  per-finding-fixer anti-pattern, measured expensive).
- CONVERGED-WITH-EXCEPTIONS is an honest terminal state: the exception
  list is part of the report and the SPEC changelog records why each
  exception exists (user said so / descoped with approval / discovered
  infeasible with evidence).

---

## 4) VERIFICATION COMMANDS

```
# V1. Convergence record exists and matches the R-count
grep -c "^| R[0-9]" <run-dir>/CONVERGENCE.md    # expect == R count in SPEC.md

# V2. Every R has evidence (file:line or command output)
grep "^| R[0-9]" <run-dir>/CONVERGENCE.md | grep -cv "implemented\|partial\|missing\|diverged\|removed"
#    expect 0 (all rows carry a verdict)

# V3. The end-to-end check was run and passed
bash <the e2e command from SPEC.md>; echo $?     # expect 0
#    paste the real output into CONVERGENCE.md - a claimed pass without
#    output is the lying-completion class

# V4. Appended tasks exist in the tasklist with R-back-references
grep -c "R[0-9]" <tasklist>                      # >= count of open items found

# V5. No out-of-scope creep
git diff --name-only <base>..HEAD | check against SPEC.md out-of-scope list
#    any overlap = finding (align the code or amend the spec IN WRITING)
```

V3 is the one that catches "all tests green, feature still does not work
end to end" - the merge-class failure that per-R greens hide.

---

## 4) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Per-R table all green, e2e fails | join-level defect; fragments pass, composition does not | e2e outranks; fix the join; re-run BOTH |
| Same R open for 3 rounds | the fix keeps being re-designed, or the R is unimplementable as written | ideas loop on that R; amend the spec in writing if the R was wrong |
| Convergence declared by reading the code, not running the verifies | verification-by-inspection | verdicts require pasted verify output; re-run |
| Converged but user finds missing work | an R was never in the spec (clarify gap) or was silently dropped | backfill the spec changelog; treat as a missed R; fix forward |
| Out-of-scope file touched "while in there" | boundary creep | Rule 0.8 finding: revert or write the spec change |
| Convergence loop never converges (infinite) | R's contradict each other or no acceptance is reachable | spec-analyze pass; resolve the contradiction at the spec level, then re-loop |
| Fixes break earlier-green R's | no regression re-run in the loop | S6 re-runs ALL R verifies, not just new ones; a regressed R reverts its verdict |

---

## 4) ANTI-PATTERNS

- THE CHECKLIST RACE: marking R's implemented by reading code paths, not by
  running the verify lines. Reading is not verification (the oldest rule in
  the fleet).
- THE SILENT DESCPE: marking a `missing` R as `removed` without a spec
  changelog line. Removal is a written decision or it never happened.
- THE CONVERGENCE SPIRAL: rounds that re-litigate closed R's while open
  ones starve. Each round prioritizes by R-order and cost; top-down.
- THE E2E SKIP: "per-R all green so e2e is implied." The e2e check exists
  precisely because parts passing individually is not evidence (verify at
  every join, not once at the end).
- CONVERGE-THEATER: one table, no evidence column, verdicts by optimism.
  The table without pasted evidence is a wish list.

---

## 5) WORKED EXAMPLES

### 5.1 The half-built feature (the common case)

Build: payment export feature, spec R1-R6. Implementer merged with tests
green. Convergence pass runs the per-R table:

- R1 (CSV export): implemented, verify run, output pasted.
- R2 (JSON behind the same interface): implemented.
- R3 (streaming for large dumps): MISSING - the code buffers in memory.
  Appended as task T9 with R3 citation.
- R4 (settings persistence): partial - settings file written but never
  read back. Appended T10.
- R5 (decrypted view rule): diverged - code exports raw dump; spec says
  decrypted. Judgment: the raw export was a deliberate safety choice by the
  implementer; spec amended (changelog: owner confirmed raw is wanted),
  R5 rewritten. Divergence resolved BY WRITING, not by silently keeping
  either side.
- R6 (deploy evidence): missing entirely - nobody deployed. Appended T11.

One more round: T9-T11 land, all R verifies re-run (R1 and R2 still green -
the regression check), e2e passes -> CONVERGED. The done-claim fires with
the full table in the report. Without the gate, R3, R4, R5 and the whole
deploy step would have been "shipped" - exactly the silent-missing-half
class.

### 5.2 The e2e-outtranks-per-R case

All 8 R verifies green. The e2e check (create order -> export -> re-import
roundtrip) fails: the export writes the new schema, the import still reads
the old one. Per-R greens were real; the JOIN was broken. UNCONVERGED, one
appended task (align the schema on both sides), re-run -> CONVERGED. This
is the "verify at every join" rule in its end-of-build form: the parts were
honest, the composition was not.

### 5.3 The honest cap

Legacy R7 ("the tool should also handle the old binary format") requires a
parser that reverse-engineering showed to be undocumented. Round 3 ends
with R7 still `missing`. Verdict: `UNCONVERGED (1 open: R7 - old binary
format undocumented; samples needed)`. The report leads with the open item
and names what closes it (sample files). No rounding, no fake
implementation, no scope-invented parser written at 2am.

---

## 6) GATES-LINE CONTRACT

Slot in the per-task extension:

```
converge: CONVERGED (R: 8/8, e2e pass) | converge: CONVERGED-WITH-EXCEPTIONS (1: <line>) | converge: UNCONVERGED (2 open: R3, R7) | converge: skipped: not a build
```

- Feeds `dim-completeness` directly: the R-table IS the completeness
  evidence; a build reporting done with UNCONVERGED status caps
  completeness at 6.0 (Unknown-class evidence per honestscore).
- The done-claim is BLOCKED while UNCONVERGED - same mechanics as the
  review gate's BLOCK. review-APPROVE + UNCONVERGED = still not done
  (quality fine, completeness open).

---

## 6) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| spec-clarify-gate | supplies SPEC.md (the checklist); converge never works from memory of the request |
| spec-analyze | catches cross-artifact contradictions BEFORE build; converge catches build-vs-spec at the end - the two ends of the same wire |
| auto-review-gate | review judges quality of the built R's; converge proves the R-set is complete; both cite the same table |
| honestscore | completeness is a dimension honestscore cannot self-supply; converge's table is its completeness evidence |
| subagents | appended tasks enter the wave plan like any task (mandate + OUT + deliverable); fix dispatches follow the review-gate who-fixes rule |
| session-monitor-auto | a convergence round with no net-new R's for 2 beats is a stall |
| ideas | open items at the cap become typed tickets |

---

## 7) OPEN ITEMS (honest limits)

- Verdict accuracy is model judgment; the verify-line requirement is the
  control (evidence-gated verdicts). Divergence judgments are the weakest
  point - require a written justification in CONVERGENCE.md for every
  `diverged`.
- The 3-round cap is budget doctrine, not a quality claim; open items at
  cap are reported and remain in the tasklist.
- No tooling beyond the markdown tables; a future pass could generate the
  table skeleton from SPEC.md mechanically (R-grep) - noted, not built.

---

## APPENDIX A - THE ASSESSMENT PACKET (what the converger reads)

The convergence pass is cheap only if the packet is assembled once:

```
1. SPEC.md                     (requirements + changelog + out-of-scope + e2e)
2. The tasklist                (what was planned vs what the waves claim done)
3. The diff surface            (git diff base..HEAD --stat + key file list)
4. Prior verdicts              (previous CONVERGENCE.md if looping)
5. The review gate's findings  (defects found there may explain missing R's)
```

The converger reads the packet, then walks R1..Rn. It does NOT re-read the
implementer's session scrollback (de-correlation rule shared with the
review gate - the table must be built from the artifact, not from the
builder's narrative).

---

## APPENDIX B - CONVERGENCE.MD TEMPLATE (copy-paste)

```
# CONVERGENCE - <task> (<date>)
status: CONVERGED | CONVERGED-WITH-EXCEPTIONS | UNCONVERGED (N open)
rounds used: <n> / 3

| R# | requirement (short) | verdict | evidence (file:line / command+output) | gap |
|----|--------------------|---------|----------------------------------------|-----|
| R1 | ...                | implemented | pytest -q tests/test_export.py -> 12 passed | - |
| R2 | ...                | missing   | grep -r "stream" src/export/ -> no hits | task T9 |

end-to-end: <command> -> <pasted output> (PASS/FAIL)
appended tasks: T9 (R3), T10 (R4) ...
out-of-scope creep check: git diff --name-only vs boundary -> clean | <finding>
exceptions: <R# - why, if CONVERGED-WITH-EXCEPTIONS>
```

---

## APPENDIX C - THE ONE-CARD OPERATING SUMMARY

```
LOAD     read SPEC.md (R-numbers, boundary, e2e check) - never memory
ASSESS   per R: locate impl -> RUN its verify -> verdict + pasted evidence
E2E      run the end-to-end check VERBATIM; it outranks per-R greens
APPEND   missing/partial/diverged -> new tasks (append-only, R-cited)
LOOP     round+1; every round needs net-new converged R's; cap 3 rounds
CLOSE    CONVERGED (all R + e2e) | WITH-EXCEPTIONS (written) |
         UNCONVERGED at cap (reported, never rounded)
CHECK    creep: diff file list vs out-of-scope boundary
NEVER    verdict-by-reading | silent divergence | decorative boundary |
         re-assessing only the new items (regressions hide there)
```

---

## APPENDIX D - EVIDENCE RULES PER VERDICT (what each verdict requires)

| Verdict | Minimum evidence to be legal |
|---------|------------------------------|
| implemented | the R's own verify line, RUN this convergence round, output pasted in the table row |
| partial | the implemented part's verify output + a precise one-line statement of what is missing |
| missing | the search that failed: the grep/recall command + empty result pasted |
| diverged | both texts side by side (spec line vs code behavior) + the written judgment |
| removed | the spec changelog line that removed it (never a silent drop) |

Evidence tiers mirror honestscore: a pasted command+output supports any
verdict; a described-but-unrun verify caps the row at `partial` (you cannot
claim `implemented` from a run you did not execute - the exit-code rule);
reasoning-only supports nothing (the row is UNVERIFIED, next round's first
job).

Consequence: a convergence table where every row has pasted evidence is the
completeness half of the done-claim, and it is auditable - anyone can
re-run any row's command and get the same verdict or a finding.

---

## APPENDIX E - THE REGRESSION RULE (why every round re-runs ALL verifies)

The convergence loop fixes missing items in rounds 2-3. A fix that breaks
an earlier-R is the classic late-stage defect (measured class: the fix
dispatch touches shared code the earlier R's depend on). Hence:

- Round N re-runs ALL R verifies, not just the newly-appended ones.
- A regressed R flips its verdict back (implemented -> partial/diverged)
  and the regression joins the appended set - with the note
  `regressed in round N by <fix task>`.
- The e2e check also re-runs every round; a green per-R table with a red
  e2e is still UNCONVERGED.

Cost note: full re-runs are cheap precisely because the R-verify lines are
single commands (spec-clarify rule). If a full re-run is expensive, that is
a spec defect (R's were written as wishes, not tests) - fix the spec's
verify lines, not the loop.

---

## APPENDIX F - CONVERGENCE INSIDE GOAL LOOPS (goal skill integration)

In a goal loop, each iteration that builds something ends with a converge
pass on THAT iteration's slice of the spec:

- The goal's DoD criteria map to R-numbers at spec-write time (the goal
  skill cites SPEC.md as its DoD source).
- The iteration report includes the converge delta: R's newly converged,
  R's regressed, open count trajectory (e.g. 7 -> 4 -> 2).
- The loop's termination ("goal met") requires CONVERGED (or honest
  exceptions) - a goal loop that ends with open spec items is not done, it
  is abandoned, and the report says which.

This gives the goal loop the completeness dimension it otherwise lacks:
honestscore judges each iteration's QUALITY, converge judges whether the
iteration moved the build toward EVERYTHING ASKED. Two different questions,
two different gates, one spec.

---

## APPENDIX G - FAQ (short, factual)

**Q: The implementer claims R3 was implemented but "needs a manual test."**
Then the R's verify line was written wrong. Either it gets an automated
verify (a test, a script), or the convergence table records
`implemented - manual verify: <exact steps>` with the steps run by the
converger or listed as the gap. "Needs a manual test" as a closing state is
a wish, not a verdict.

**Q: Divergence where BOTH readings are defensible?**
That is a requirements ambiguity that clarify missed. The converger picks
neither - it writes the conflict and routes it to the clarify escalation
path (one question, recommended option). Divergence with a real product
decision inside it is an owner decision, not a convergence decision.

**Q: Can converge run BEFORE review?**
Yes - converge is completeness, review is correctness; both run before the
done-claim. Order by cost: converge (table-driven, fast) then review
(fresh-context) on the converged artifact. Running review on an
unconverged build wastes the review on artifacts that are about to grow.

**Q: Single-file small build - full procedure?**
The procedure compresses (the packet is 2 files, the table is 3 rows) but
never skips: the R-table + e2e run is the whole point. A 3-row table takes
minutes and catches the missing-half class.

**Q: What if SPEC.md changed mid-build (legitimately)?**
The changelog governs. Convergence assesses against the CURRENT spec
version; R's removed before convergence rounds started do not block. An R
removed AFTER an open finding cited it needs the changelog to say why - the
converger checks the changelog before marking anything.

---

## APPENDIX H - CREW MERGE CONVERGENCE (subagents integration)

For a multi-seat build, convergence runs on the MERGED artifact and the
per-R table doubles as the coverage check the merge step needs:

- Each R cites the fragment (seat/task id) that fulfilled it - an R with NO
  owner fragment after merge is the unassigned-remainder defect, caught
  here instead of at the user.
- An R claimed by TWO fragments (double-implementation) is a finding in
  its own right (duplicate scope - the subagents Rule 2 defect class);
  the converger marks it and the merge resolves by evidence (pasted output
  wins), per the subagents merge rules.
- The appended-tasks list feeds wave 2+ of the SAME run (not a new run) -
  convergence is part of the run's wave plan, not a post-run audit.

Practical order for a crew build: merge (orchestrator) -> join reviews
(auto-review-gate Appendix J) -> converge pass 1 -> appended fixes as a fix
wave -> re-converge -> full review -> done-claim. Each stage is cheap
because the previous one narrowed the surface.

---

## APPENDIX I - ANTI-CHEAT: THE KNOWN GAMING PATTERNS

Convergence gates attract the same gaming as any gate; the known moves and
their counters:

| Gaming move | Counter |
|-------------|---------|
| Mark `implemented` on a verify that was never run this round | the pasted-output rule; converger re-runs any row it did not witness |
| Weaken the R's verify line mid-loop so it passes | verify-line edits are spec changelog events; diff the SPEC between rounds |
| Split one R into two `partial` rows to halve the gap | one R = one row; a split R is a spec edit, decided once |
| Declare e2e "implicitly covered by per-R greens" | the e2e line exists because this claim is false (Appendix 5.2) |
| Remove the blocking R from the spec to converge | changelog must carry the why; blocking-R removal mid-loop = escalate to owner, not converge |
| Converge against a PARAPHRASED spec from memory | the packet loads the file; `converge: spec not loaded` is the failure mode |

The meta-rule: every counter is a FILE check (diff, grep, paste), not a
trust decision. The gate holds because it never has to believe anyone.

---

## APPENDIX J - METRICS (one line per convergence pass)

```
converge | <date> | spec <path> | R total/open/missing/partial/diverged
| rounds <n> | e2e <PASS/FAIL> | appended <count> | regressions <count>
| creep <clean|finding> | verdict <CONVERGED|W-EXC|UNCONVERGED>
```

Signals: open-count trajectory across rounds (should be strictly
decreasing; flat = stall); divergence rate (high = clarify is under-class);
regression count (nonzero = the fix dispatches need the review gate's
who-fixes rule enforced harder). Feeds the same calibration dataset as the
review and clarify metrics - one file, three gates.

---

## APPENDIX K - FULL WORKED PASS (annotated end to end)

Scenario: crew-built "client report exporter" (3 seats, merged), SPEC.md
has R1-R8, this is convergence round 1.

```
Step 1 - packet loaded
  SPEC.md @ v3 (changelog shows R6 reworded after clarify round 1)
  diff surface: 14 files vs base 9f2c1a
  boundary: "no PDF export; no scheduler" (out-of-scope list)

Step 2 - per-R pass (excerpts)
  R1 CSV export        | implemented | python -m pytest -q tests/test_csv.py
                       |   -> 9 passed                         [pasted]
  R2 JSON same iface   | implemented | tests/test_json.py -> 6 passed  [pasted]
  R3 streaming >100MB  | missing     | grep -rn "chunk" src/export/ -> no hits
  R4 settings persist  | partial     | write path exists (settings.py:31);
                       |             | read-back untested/no code found
  R5 decrypted view    | diverged    | spec: decrypted | code: raw export
                       |             | -> conflict written, routed to owner
  R6 audit log line    | missing     | no audit module in diff
  R7 admin gate        | implemented | middleware registered (app.py:88) [pasted]
  R8 error banner i18n | implemented | banner key test passes           [pasted]

Step 3 - e2e (verbatim from spec)
  $ python scripts/e2e_export.py
  ... export ok, re-import ok, checksum match
  PASS [pasted]

Step 4 - creep check
  git diff --name-only | vs boundary: clean (no scheduler files touched)

Step 5 - append + verdict
  appended: T7 (R3 impl+verify), T8 (R4 read-back path), T9 (R5 blocked on
            owner decision - escalated, not appended as work)
  verdict: UNCONVERGED (2 implementable open, 1 owner decision open)

Round 2 (after fix wave): R3/R4 converge with fresh verify outputs; R5
  resolved by owner answer -> implemented; e2e re-run PASS; R1/R2/R7/R8
  re-run green (no regressions). verdict: CONVERGED (rounds 2/3).
  Then - and only then - the full review gate runs on the converged build.
```

The pass cost two tables and a handful of commands. The alternative -
trusting the implementers' green self-reports - is the half-shipped
feature this skill exists to prevent.

---

## APPENDIX L - CONVERGENCE FOR NON-CODE ARTIFACTS (docs, skills, sites)

Completeness applies to any artifact class with a spec:

- SKILL FILES: R-lines become the required sections (purpose, fire, steps,
  verify commands, failures, gates-line, example) + line count + ASCII +
  frontmatter. The e2e check is the mechanical scan (frontmatter parse +
  ASCII count + line count in range). Converge catches "the skill exists
  but has no verification section" - the equivalent of a half-shipped
  feature.
- CLIENT SITES: R-lines = the page list with per-page acceptance (renders,
  mobile, a11y, perf budget), e2e = crawl of the sitemap with real browser
  checks. Converge catches the built-5-of-7-pages class.
- RUNBOOKS / REPORTS: R-lines = each section the task demanded; e2e = the
  section-count/traceability check (every claim sourced).

The machinery (verdicts, append-only loop, cap, changelog) is identical
across artifact classes - only the verify-line shape changes. One
procedure, any artifact.

---

## APPENDIX O - EXTENDED TROUBLESHOOTING, PART 2

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Two converge passes disagree on the same build | different spec versions loaded | both passes log the SPEC version hash in the verdict; disagreement across versions is not a conflict, re-run on current |
| Converger "can't find the implementation" for an R that exists | R written in UI language, code uses domain terms | the R's verify line must name the command that locates it; fix the spec line (changelog) and re-assess that row |
| Convergence loop ping-pongs R4 open->partial->open | the "fix" toggles behavior | treat as a diverged-fix: evidence in both rounds, escalate like a divergence |
| Done-claim fired despite UNCONVERGED | gate bypassed silently | the once-per-rollout bypass test (review gate V3 analog): deliberately leave an R open, confirm the done-claim is refused; if not, the gate does not exist |

### Part 1 - the core classes

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Table says all-green but the feature is obviously absent | assessed against a paraphrased spec from memory | reload SPEC.md; the R-grep (`grep -c "^R[0-9]"`) is the anchor |
| Verify command passes locally, fails in CI/converger | environment drift (deps, cwd, env vars) | verify lines must be self-contained (spec-clarify rule); converger runs them in the build's own venv/venv-less harness |
| Divergence judgment flips between rounds | no written justification persisted | every `diverged` row carries the both-texts excerpt + the routed decision; the record is the judgment |
| Appended tasks never enter any wave | fix list produced, never dispatched | appended tasks enter the tasklist like any task; the orchestrator owns dispatch (footprint check on completion claims) |
| Convergence "passes" with the owner's pending decision unresolved | verdict declared to close the session | owner-decision R's force `UNCONVERGED` or explicit `CONVERGED-WITH-EXCEPTIONS` naming the decision - never a silent skip |
| Regression found in round 3, budget exhausted | fix broke earlier R | honest report: verdict UNCONVERGED (regression), revert-or-fix is the next action, budget extended only by owner say-so |
| The e2e check is itself broken (fails for env reasons) | spec wrote an unrunnable e2e | fix the e2e check (spec changelog), do not downgrade the gate; an unrunnable e2e blocks CONVERGED by definition |

## APPENDIX M - CRASH RESUME (convergence state is durable)

A convergence pass interrupted mid-loop resumes from files, not memory:

### State files

```
<run-dir>/CONVERGENCE.md      <- verdicts so far, rounds used
<run-dir>/SPEC.md             (versioned; changelog authoritative)
<run-dir>/tasklist.md         (appended tasks carry R-citations)
```

Resume rule: re-run the CURRENT round from its start (re-assess all R's) -
partial rounds are discarded, because a half-filled table with stale
verdicts is worse than a clean re-run. The cap counts ROUNDS EXECUTED with
a written verdict line, so a crashed round does not consume budget.

The round's written verdict line format (top of CONVERGENCE.md, updated in
place):

```
round 2 status: ASSESSING | APPENDING | CLOSED <verdict> - updated <ts>
```

The watchdog reads exactly that line: a convergence pass sitting in
ASSESSING for 2 beats with no table growth is a stall (session-monitor-auto
integration, same beat rule as every other long step).

---

## APPENDIX N - CONVERGE + REVIEW + HONESTSCORE: WHO OWNS WHAT

The three end-of-build gates answer three different questions; keeping them
distinct prevents both duplication and gaps:

| Gate | Question | Output |
|------|----------|--------|
| spec-converge | is everything ASKED built? | per-R table, e2e, appended tasks |
| auto-review-gate | is what was built CORRECT (and scoped)? | severity-graded findings, APPROVE/BLOCK |
| honestscore | what is the honest quality score? | MIN across dimensions, defect tickets |

Order: converge -> review -> score. Converge first so review judges a
complete artifact (reviewing an about-to-grow build wastes the fresh
session); score last so its completeness dimension cites the converged
table.

Feeding: converge's open items become review scope additions (an open
`partial` R tells the reviewer where to look hardest); review findings on
missing verify commands become spec fixes (changelog); honestscore's
completeness dimension cites the final table. One pipeline, three lenses,
zero double-judging.

---

## APPENDIX P - WORKED NON-CODE PASSES (short, real shapes)

### P.1 Skill file convergence (this very authoring run's pattern)

R-lines (from the skill-file spec): purpose, when-to-fire, procedure,
verification commands, known-failure table, gates-line definition, worked
example, 800-3,000 lines, ASCII-only, frontmatter complete.

```
| R purpose    | implemented | section present (grep '^# ')            |
| R fire       | implemented | "## 1) WHEN IT FIRES" present           |
| R verify     | implemented | Section 6 with runnable commands        |
| R line-count | implemented | wc -l -> <n> (in 800..3000?)            |
| R ascii      | UNVERIFIED  | scan pending                            |
e2e: frontmatter parse + R-grep + ascii scan -> run at close
```

That UNVERIFIED row is the point: an authoring run that skips the ASCII
scan ships a skill that crashes some runtimes - the converge pass exists
precisely so no row is still pending at close time.

### P.2 Client-site convergence (per-page acceptance)

R1-R7 = seven pages; each R's verify = browser check (render, mobile
viewport, a11y pass, LCP budget); e2e = full sitemap crawl. Typical
findings: one page never built (appended), a mobile breakpoint diverged
(design decision routed to owner), e2e catching the broken nav link that
no single-page check sees.

### P.3 Report convergence (traceability)

R-lines: "every claim sourced", "fetch dates on drift-prone facts",
"NOT-verified list present", "no secrets". e2e = the source-count check
(every R cites at least one tool result). The half-sourced report is the
doc-shaped version of the half-shipped feature - same defect class, same
gate.

The three share the exact machinery of Appendix K: verdicts, append-only
loop, cap, changelog. One procedure, any artifact - which is why the skill
stays generic instead of per-artifact.

---

## APPENDIX Q - THE MINIMUM VIABLE CONVERGENCE (near budget)

When the run is near budget and the full table would not fit, the MINIMUM
legal pass is:

```
converge (minimum):
  R-count assessed: <n>/<total>   (must be ALL - no sampling)
  e2e: <command> -> <output> (PASS/FAIL)
  creep check: diff file list vs boundary -> clean|<finding>
  verdict: <CONVERGED|W-EXC|UNCONVERGED (open: R3,R6)>
```

Not droppable even at minimum: the e2e run, the creep check, honest
verdicts on every R. What compresses: for R's whose verify already ran in
an earlier wave of THIS run, cite the wave output marked `witnessed`
instead of re-running. What never compresses: the verdict-by-reading ban,
the append-only tasklist, the changelog on any spec edit.

---

## APPENDIX R - WHY "CONVERGE" (the name)

The Spec Kit lineage calls the final state "Converged": the plan, the
tasks, and the code have converged on the same meaning. The loop is
arithmetic, not aspirational - each round reduces the distance between the
spec and the artifact, measured in open R's, until the distance is zero
(or the honest exceptions are written).

also be complete and poor (CONVERGED, honestscore 6.1). Both gates must
pass for the same done-claim - that is the whole design.

---

## APPENDIX S - GATES-LINE AND REPORTING EXTENSION

The per-task gate extension slot for this skill:

```
converge: CONVERGED (R 8/8, e2e PASS, rounds 2/3)
converge: CONVERGED-WITH-EXCEPTIONS (R5: owner chose raw export)
converge: UNCONVERGED (2 open: R3, R6) - honest, not rounded
converge: skipped: not a build | skipped: handoff already converged
```

Values never contain the pipe character. A skipped convergence on a build
task is a gate bypass and gets counted by the audit (same bypass ledger as
the review gate). The completion report pastes the final table (or the
Appendix Q minimum block) - a converge claim without a pasted table is a
lying completion report by fleet rule.

Dim feeding: `dim-completeness` cites the final table; `dim-docs` cites
the changelog when spec edits occurred mid-build (an undocumented spec
edit is a docs finding).

---

## APPENDIX T - TEN-SECOND SELF-CHECK (run before any convergence claim)

Answer these five with file evidence; any "no" means the pass is not done:

1. Did the e2e command run THIS pass with its output pasted?
2. Does every R row carry a verdict AND an evidence pointer?
3. Does the appended-task list exist in the tasklist (not just in prose)?
4. Did the out-of-scope creep check actually diff the file list?
5. Is the verdict line written (CLOSED / UNCONVERGED) in CONVERGENCE.md?

The checklist exists because the most common convergence failure is the
verdict declared two steps before the evidence existed - the same
completion-vs-evidence inversion the anti-overclaim doctrine bans.
