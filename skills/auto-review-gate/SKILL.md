---
name: auto-review-gate
description: Mandatory fresh-context blocking reviewer on every diff before any done-claim. The reviewer NEVER saw the build (no shared session), is scoped to correctness + the stated requirements only, grades findings by severity (Critical/Major/Minor/Info), and BLOCKS the done-claim while any Critical or Major finding is open. Re-verifies fixes in a fresh session before unblocking. Warning-mode first, promote to blocking after 3 clean runs (the CodeRabbit rollout pattern). Closes the generator-judges-itself gap (A04, master upgrade table, P2; ranked #9 by impact on 20h+ autonomous runs). alwaysFire - fires before every done/fixed/shipped claim on any artifact that produced a diff.
alwaysFire: true
---

# auto-review-gate - the fresh-context blocking reviewer

You are running **auto-review-gate**. A diff exists (any artifact change: code,
docs, config, prompt, skill). A done-claim is about to be made. Between those
two moments stands this gate: a reviewer who (1) never saw the construction,
(2) may not edit, (3) is scoped narrowly enough that it does not invent
defects, and (4) can actually block.

**The measured failure this closes:** the generator-judges-itself gap. The
owner's rule, measured repeatedly: re-reading your own answer 6-8 times mostly
re-confirms it, because the same reasoning that produced the error produces the
same review. What catches errors is a check that could have come out the other
way - and a fresh-context reviewer is the strongest such check that exists for
a diff. The honestscore rubric independently requires a de-correlated
adversarial pass; this skill is the STANDING procedure that makes it
automatic, severity-graded, and blocking instead of ad hoc.

**What this skill is NOT:**

- NOT a second opinion on taste. The reviewer's mandate is correctness +
  the stated requirements - nothing else. An unscoped "find problems" pass
  invents defects and drives over-engineering (measured: reviewer agents told
  to "find problems" fabricate over-engineering demands).
- NOT a style gate. Lint/format belong to the language gates (ruff, gofmt,
  dotnet format) which run mechanically already.
- NOT optional. On any done-claim without a completed review record, the
  claim is premature by definition - the gates line reports
  `review: skipped: why` and honestscore caps at 7.0 (its own rule: no
  de-correlated adversarial pass = NOT-MET gate -> cap 7.0).

---

## 0) HARD RULES (survive compaction)

1. **Fresh context is structural, not attitudinal.** The reviewer is a NEW
   subagent session that receives ONLY: the diff (or artifact), the original
   requirement text, and the review rubric. Never the implementer's session,
   never its reasoning, never its claims. Reusing the writer's session voids
   the pass - de-correlation is the entire point.
2. **Read-only reviewer.** The review seat gets no Write/Edit tools (or a
   hard instruction + agent definition that denies them). A reviewer that can
   fix is a second implementer, and its findings stop being independent.
3. **Blocking on Critical/Major only.** Minor and Info findings are logged,
   not blocking. Warning-mode deployments block nothing; they log (Section 6).
4. **Every finding cites evidence**: file:line, a command output, or a spec
   sentence. A finding without an anchor is dropped (reviewers that reason
   from memory produce the fabricated-defect class).
5. **Fixes ship as ONE dispatch.** All open Critical/Major findings go back in
   a single fix dispatch (per-finding fixers are the measured anti-pattern).
   Fix dispatches 1-2 resume the original implementer; 3+ = fresh implementer
   briefed with the prior attempts report.
6. **Re-verify in a fresh session, hard cap 2 rounds.** Findings still open
   after 2 re-verify rounds are reported honestly as open, never rounded
   away. A third silent "re-verify" is banned.
7. **The review verdict is a record, not a feeling.** Every review writes a
   findings block (Section 4 format) to `<run-dir>/review-<n>.md`. The
   gates line's review slot cites that file. No file, no review.
8. **The implementer writes its own score FIRST.** Before the reviewer
   returns, the implementer writes its own expected-verdict line to the run
   dir (anchor + compare; the honestscore Step 0 pattern applied to review).
9. **No review of its own review.** If a re-review is needed after fixes, it
   is a NEW session again. The same reviewer session that was overruled will
   rubber-stamp its own follow-up.
10. **Scope statement travels with the review.** "Scoped to: correctness +
    requirements R1-R4" is part of the record. A review that drifted to style
    or scope-invention is invalid and is re-run with the scope line enforced.

---

## 1) WHEN IT FIRES

Always-fire BEFORE any done-claim, on ANY artifact change:

| Trigger | Review scope |
|---------|--------------|
| Any code diff (staged, unstaged, or worktree diff vs base) | the diff + its callers (regression cone) |
| Any doc/skill/prompt file written or edited | the file vs its stated purpose |
| Any config change (JSONC/YAML/env) | the change + both-config-root law check |
| Any deploy | the deploy evidence chain (what proves it works) |
| Merge of subagent fragments | the MERGED artifact (per-fragment pass is not enough) |
| Brain note / knowledge file update | factual claims vs sources |

Skip conditions (honest, reported): `trivial ask` (one-line conversational
answer, no artifact), `no diff this turn` (read-only work - say so). There is
no "small diff" skip: the one-line fix is exactly where the confident error
lives. A one-line diff gets a ONE-QUESTION review, not zero questions.

---

## 2) THE REVIEWER SPAWN CONTRACT

### 2.1 What the reviewer receives (complete list - nothing else)

```
1. THE ARTIFACT: the diff itself (`git diff <base>..<head>` output pasted
   in full, or the file content for new files), plus - for context only -
   the surrounding functions it touches.
2. THE REQUIREMENTS: the original task statement, verbatim, including the
   success criteria and explicit out-of-scope list. (No requirements were
   written? That is a spec-clarify-gate defect - review cannot proceed and
   reports exactly that.)
3. THE RUBRIC: the finding format + severity definitions (Section 3) +
   the scope line: "scope = correctness + stated requirements. Do NOT
   comment on style, naming preferences, architecture alternatives, or
   improvements nobody asked for."
4. THE EVIDENCE RULE: every finding cites file:line or a pasted tool result;
   findings must be executable checks or direct reads, not reasoning-only.
5. The honestscore Step 0b instruction: restate each load-bearing claim as
   a question before judging it.
```

### 2.2 What the reviewer NEVER receives

- The implementer's session, transcript, or self-assessment (de-correlation).
- The implementer's honestscore self-score (that comparison happens AFTER,
  and the lower number wins).
- Previous review rounds' findings (a re-review after fixes gets: the
  artifact, the requirements, AND the specific findings to re-verify - but
  the reviewer is still a fresh session and still reads the artifact itself).

### 2.3 The reviewer's required output (parsed deterministically)

```json
{
  "verdict": "APPROVE | BLOCK",
  "scope_statement": "correctness + requirements R1..Rn",
  "findings": [
    {"severity": "Critical|Major|Minor|Info",
     "file": "path", "line_or_location": "...",
     "claim": "one sentence",
     "evidence": "file:line | pasted output | command",
     "required_fix": "one sentence"}
  ],
  "requirements_coverage": {"R1": "met|unmet|unverifiable", "...": "..."}
}
```

BLOCK = any Critical or Major present. The orchestrator does not get to
disagree with BLOCK by mood - only by fixing or by producing the evidence
that the finding is false (with the evidence pasted).

---

## 3) SEVERITY GRADES - the exact ladder

| Severity | Definition | Examples | Gate effect |
|----------|-----------|----------|-------------|
| **Critical** | requirement violated, data loss/corruption, security hole, or a claim that is false | test weakened to pass; auth check skipped on one path; secret committed; the "verified" claim with no run | BLOCK |
| **Major** | correctness bug reachable in normal use, or missing stated requirement | unhandled realistic input; race on shared state; edge case in the spec missed | BLOCK |
| **Minor** | real defect, narrow blast radius, or robustness gap unlikely in practice | typo in a log string; missing retry on a best-effort call | logged, non-blocking |
| **Info** | observation, question, or note - not a defect | "this helper is now used twice" | logged, non-blocking |

Anti-inflation rules for the reviewer:

- A "finding" that restates the requirement as unmet WITHOUT evidence is Info,
  not Major.
- A performance concern with no measurement is Info (measurement-discipline:
  "should be faster" is not a finding, it is a hypothesis).
- A "missing feature" that is in the stated out-of-scope list is DROPPED and
  logged as scope-creep attempt (this is the over-engineering guard).
- Maximum findings per review: 12. More than that means the reviewer is
  listing style preferences - it must cut to the top by severity, rest Info.

---

## 4) THE PROCEDURE (step by step)

**S1. Freeze the artifact + write the anchor.** Implementer writes the
expected-verdict line to `<run-dir>/review-anchor.md`: "expected verdict
APPROVE; risks I know about: <list>". One or two lines. This is the
commit-before-look step - it makes drift between expectation and review
visible instead of rationalized.

**S2. Assemble the review packet.** Diff (or artifact text) + requirements +
out-of-scope list + rubric (Section 2.1). Nothing else. No session history.

**S3. Spawn the fresh reviewer.** One subagent, read-only, most capable model
reasonable for the artifact size. Mandate (exact shape):

```
You are a blocking code reviewer. Scope: correctness + the stated
requirements ONLY. You did not see this work being done and you have no
access to the author's reasoning - read the artifact itself.

Build the case this diff FAILS. Read/execute what you need; every finding
must cite file:line or a pasted tool result. Grade each finding
Critical/Major/Minor/Info per the ladder. Check requirement coverage
explicitly (R1..Rn). You have no write access and you do NOT propose
rewrites - findings only, one required action each.

If you cannot demonstrate a defect with evidence, it is not a finding.
```

**S4. Parse the verdict.** APPROVE -> record + proceed to done-claim (the
gates line cites `review: APPROVE (findings M/I, 0 blocking)`). BLOCK ->
assemble the open Critical/Major findings into ONE dispatch.

**S5. Fix round.** All findings, ONE dispatch (Section 0 rule 6). Who fixes:
1-2nd dispatch resumes the original implementer (context intact); 3rd+ goes
to a fresh worker briefed with "a prior implementer attempted this; read
<report path> for what was tried and what failed."

**S6. Re-verify.** NEW reviewer session (never the same one; never the
writer). The re-review receives the previous findings list + the updated
artifact, and must independently confirm each fix by reading/running - an
acknowledgment is not a fix, and a fix that re-introduces the defect (the
measured class) is caught only by an independent re-read.

**S7. Loop or land.** Hard cap: 2 re-verify rounds. Findings still open after
them are reported honestly in the done-claim ("open: Major#2 - <finding>"),
never rounded away. The record stays in the runlog.

---

## 5) WARNING-MODE ROLLOUT (new installs)

Blocking review with no warm-up creates review fatigue and gaming (rubber-
stamp APPROVEs). Roll out like CodeRabbit did:

| Phase | Duration / trigger | Behavior |
|-------|--------------------|----------|
| 0. Shadow | first 5 reviews | findings logged only; measure reviewer precision (are findings real?) |
| 1. Warning | next 5 or until 3 clean reviews | findings logged + surfaced in the report; done-claim allowed with findings listed |
| 2. Blocking | steady state | Critical/Major block the done-claim |

Promotion is earned: 3 consecutive reviews with zero Critical/Major findings
in the same artifact class, and no fabricated-defect pattern in Minor/Info.
Demotion (back to warning) happens on 2 consecutive reviews with >= 2 dropped
false-Critical findings - a reviewer that cries wolf gets demoted, the same
honesty rule as every other gate.

Record phase in the review record: `mode: shadow|warning|blocking`.

---

## 6) VERIFICATION COMMANDS (the gate on the gate)

```
# V1. The record exists and is fresh
ls -la <run-dir>/review-*.md            # newest mtime AFTER the last artifact edit
#    (a review older than the last edit is reviewing a stale artifact - re-run)

# V2. The reviewer was fresh (no shared session)
#    record shows reviewer session-id != implementer session-id

# V3. Blocking findings actually blocked (guard-proves-itself test)
#    once per rollout: deliberately leave one Major open, confirm the
#    done-claim was refused / honestscore capped at 7.0 (no review pass)
#    If the pipeline let it through, the gate does not exist.

# V4. No writer drift: post-review artifact hash unchanged since review
git rev-parse HEAD | diff - <run-dir>/review-<n>.head   # or hash the file
#    any edit after APPROVE re-opens the review (subagents S7 freeze rule)

# V5. Scope kept: the record's scope_statement line matches the task's
#    stated requirements + out-of-scope list (drift = invalid review)
```

V3 is the one most installs skip: a review gate that has never refused a
done-claim has never proven it can. Attack your own gate.

---

## 7) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Reviewer APPROVEs everything | writer's context leaked in, or rubric absent | re-check the packet: artifact + requirements + rubric ONLY; re-run fresh |
| Reviewer invents style defects | scope line missing or soft | enforce the scope line verbatim; drop non-correctness findings to Info and log the drop |
| Same finding reopens after "fix" | fix was a suppression (test deleted, check weakened) | re-verify pass must diff the FIX not just re-run; a deleted test is a Critical finding on its own |
| Review of a merged crew artifact only checked one fragment | reviewer got fragments, not the merge | review packet = the MERGED artifact only (subagents S7); reject per-fragment reviews at joins |
| Review blocks forever on a subjective finding | severity misgraded | pre-mortem: "would this fail the user's stated success criteria?" - if no, it is Minor |
| Two reviewers disagree wildly | mandates differed | one rubric, one scope statement, versioned; disagreement is resolved by evidence (pasted output wins) |
| Review skipped "because trivial" | skip condition abused | the one-question review for one-line diffs (S1 skip rules); silent skip is the violation |
| Reviewer has no tool access and reasons from memory | spawn config wrong | read-only agent WITH Read/Bash; "execute/read the artifact" is in the mandate |

---

## 8) ANTI-PATTERNS

- THE RUBBER-STAMP: review that exists to be reported ("review: done") with
  no findings and no evidence. A review with zero findings on a non-trivial
  diff is suspect by default - the record must show WHAT was checked.
- THE REWRITE REQUEST: reviewer proposing its own implementation. Findings +
  one-line required actions; the fix belongs to the implementer.
- THE FRIENDLY REVIEWER: same session, same context, "review" of its own
  work. Void by definition (Rule 0.1).
- THE UNBOUNDED REVIEW: "find everything wrong." Invents defects, drives
  over-engineering. Scope = correctness + stated requirements, in writing.
- THE PERPETUAL WARNING MODE: warning-mode that never promotes. Promotion
  trigger is arithmetic (3 clean runs), not a mood.
- REVIEW-THEATER ON GENERATED FILES: lockfiles, generated output, build
  artifacts are excluded from review scope; reviewing them is noise. State
  the exclusion in the packet.

---

## 9) WORKED EXAMPLES

### 9.1 The gates-line format drift (real campaign case, replayed)

Diff: sessions began emitting a 4-5-field gates line instead of the canonical
14. The implementer's self-check read "gate line present -> pass."

Under this skill, the fresh reviewer reads the format spec, asks Step 0b
("does a 4-field line satisfy the 14-field requirement?"), pastes both lines
side by side -> **Major: format drift, 80-85% strict compliance, spec says
14 fields**. BLOCK. The fix (v14.2 plugin injecting the full template) was
found by the fix dispatch; the re-review in a fresh session measured the
strict-14 rate on post-fix sessions before APPROVE. The generator's self-
check could never have found it - it was checking the wrong field count and
confirming itself.

### 9.2 A one-line config fix

Change: one timeout value 30 -> 120 in a retry helper. The one-line review:
packet = the diff + the requirement "long jobs must not time out" + the
caller list. Reviewer traces callers (git grep the key), finds one consumer
that treats >60s as dead-peer and marks it failed -> **Major** (correctness
for that caller), BLOCK, one-line required action: "bump the consumer's
threshold or split the keys." One-question review, ten minutes, one
production incident prevented. Small diffs get small reviews, not no reviews.

### 9.3 The merge of a 6-seat crew build

Each seat's fragment passed its own check. The merged artifact had two
fragments using different names for the same config key (semantic conflict,
compatible lines - git saw nothing). The post-merge review, scoped to "does
the assembled deliverable satisfy the spec", reads the COMPOSED artifact,
runs its acceptance commands, and catches the key mismatch at the join
(subagents S7: verify at every join). Per-fragment reviews had each said
APPROVE; the join review is the one that mattered.

### 9.4 Honest skip

A read-only research turn ends with a report. No artifact changed, so the
gates line reads `review: skipped: no diff this turn` - honest and correct.
The NEXT turn, when that report gets edits, the review fires on the edited
files only, with the original requirements as the scope statement.

---

## 10) GATES-LINE CONTRACT

Slot in the canonical 14-field line: this skill feeds
`dim-correctness` (review evidence = pasted findings + verdict) and reports
its own run status in the per-task extension slot:

```
review: BLOCKING APPROVE (crit 0, major 0, minor 2) | review: BLOCK (1 critical: <one line>) | review: skipped: <why>
```

Honest skip whys: `skipped: no diff this turn`, `skipped: trivial ask`,
`skipped: no spawn tool - self-reviewed with scope line, 0 findings`
(degraded self-review is allowed ONLY when spawning is impossible; it caps
honestscore's correctness dimension per its Unknown rule - say so).

Interaction with honestscore: the reviewer pass and the honestscore
adversarial pass may be the SAME spawn when both are required (the rubric is
included in the reviewer packet) - one de-correlated pass feeding both
records, as long as the reviewer's verdict and the score's MIN are both
reported. Do not run two identical fresh-context passes on a small diff;
that is the over-engineering the scope line exists to prevent.

---

## 11) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| honestscore | this skill's BLOCK pass IS the honestscore adversarial gate when both fire on a done-claim; the reviewer gets the rubric explicitly (subagents S8 rule) |
| subagents | reviewer seats spawn AFTER merge, never with workers; reviewer counts against the 30-cap |
| ideas | reviewer findings are typed tickets (defect | cost | closing-evidence); ideas works them highest-cost first |
| anti-overclaim | the done-claim vocabulary gate runs first (phrase check), then this review runs on substance |
| verification-before-completion | the implementer's own checks run BEFORE review; review is not a substitute for running the build/tests |
| session-monitor-auto | a reviewer that stalls (no findings after 2 beats on a real diff) is chased like any other child |

---

## 12) OPEN ITEMS (honest limits)

- Reviewer quality is model-bound; the packet reduces (not eliminates) the
  fabricated-defect class. Measure: track findings that were closed as
  not-a-defect; >30% fabricated = demote to warning and re-scope.
- No automated severity classifier: grading is the reviewer's judgment under
  the ladder. A severity-appeal goes to a second fresh reviewer only for
  Critical disputes (cost control).
- The 2-round cap on re-verification is a budget rule, not a quality claim;
  open findings at the cap are reported, never rounded away (honestscore
  carries them as open tickets).
- Metrics recorded per review feed the rollout arithmetic and judge
  calibration; a review system that measures nothing cannot be promoted -
  and cannot be trusted once it is.

---

## APPENDIX H - ONE-CARD OPERATING SUMMARY (reference card)

```
FIRE     before EVERY done-claim on any artifact change (no size skip)
FRESH    new subagent session; artifact + requirements + rubric ONLY;
         no writer reasoning, no writer session, read-only tools
SCOPE    correctness + stated requirements. Nothing else. In writing.
GRADE    Critical (block) | Major (block) | Minor (log) | Info (log)
         <=12 findings; everything else dropped to Info with the drop logged
BLOCK    any Critical/Major open -> BLOCK -> ALL findings in ONE fix dispatch
         (1-2nd fix: resume implementer | 3rd+: fresh worker + prior-attempts file)
REVERIFY fresh reviewer, new session, independently confirm each fix;
         hard cap 2 rounds; open findings reported honestly, never rounded
ROLLOUT  shadow x5 -> warning x5 (or 3 clean) -> blocking; demote on 2
         reviews with >=2 fabricated Criticals
RECORD   review-<n>.md + runlog metric line; review older than the last
         artifact edit is void; edit-after-APPROVE reopens the review
SELF     V1 record fresh | V2 session de-correlated | V3 gate refuses a
         deliberately-open Major once per rollout | V4 hash freeze |
         V5 scope statement matches the task
NEVER    rubber-stamp | rewrite requests | friendly (same-session) review |
         unbounded "find everything" scope | review of generated files
```

---

## APPENDIX A - THE REVIEW PACKET TEMPLATE (copy-paste)

Every review spawn receives EXACTLY this packet (fill the brackets):

```
=== REVIEW PACKET v1 ===
mode: shadow | warning | blocking
artifact_type: code | config | doc | prompt | skill | merge
scope_statement: >
  Review scope = correctness + the stated requirements ONLY.
  Requirements:
    R1: <verbatim from the task>
    R2: <verbatim>
  Out of scope (do NOT flag as missing):
    <verbatim out-of-scope list>
    style/naming preferences; generated files (<list>); lockfiles.
diff_or_artifact:
  <git diff base..head -- output, or full file text for new artifacts>
  <surrounding context: the enclosing functions, cited with file:line>
reviewer_rules:
  - Every finding cites file:line OR pasted tool output.
  - Severity ladder: Critical (requirement violated / false claim /
    security) | Major (reachable correctness bug / stated requirement
    missed) | Minor (real, narrow) | Info (note, not a defect).
  - Max 12 findings; overflow drops to Info by severity.
  - You have NO write access. Findings + one-line required actions only.
  - Step 0b: restate every load-bearing claim in the artifact as a
    question before judging it.
  - Requirements coverage table is mandatory: R1..Rn -> met|unmet|unverifiable.
verdict_format: APPROVE | BLOCK (BLOCK iff any Critical or Major)
```

### A.1 Packet assembly commands (implementer runs these)

```
# code diff
git diff <base>..HEAD > <run-dir>/packet-diff.txt
# context for touched hunks (the enclosing function bodies)
git diff <base>..head -U12 -- <files> >> <run-dir>/packet-diff.txt
# requirements + out-of-scope: copy from the task spec file, verbatim
# rubric: paste Sections 3 + 2.3 of this skill, verbatim
```

The packet is assembled ONCE per review. Re-packing mid-review (adding
context the reviewer "should also see") voids the review.

---

## APPENDIX B - FINDING RECORD FORMAT (typed tickets for /ideas)

Every finding in `review-<n>.md` uses this exact shape:

```
### F<n> | <severity> | <file:line or location>
claim: <one sentence, demonstrable>
evidence: <pasted tool result / command + output / spec quote>
required_action: <one line>
point_cost: <0.1-0.3 cosmetic | 0.4-0.8 moderate | 1-2 serious>
closing_evidence: <the check that proves the fix>
```

The last two fields make findings directly consumable by ideas (typed
tickets) and honestscore (dimension ledgers). A finding missing
`closing_evidence` is returned to the reviewer to complete - untestable
findings are not findings, they are opinions.

Example, fully filled:

```
### F3 | Major | src/retry.py:41
evidence: python -c "import retry; retry.call(flaky, tries=3)" -> RuntimeError
          after 3 attempts (expected: backoff between attempts; log shows
          zero delay: ts 09:14:02.001, 09:14:02.003, 09:14:02.004)
required_action: sleep backoff_ms between attempts, not only after the first
point_cost: 1.0
closing_evidence: re-run with timestamps showing >=100ms gaps; unit test
                  with a fake clock
```

---

## APPENDIX C - REVIEW MODES PER ARTIFACT TYPE

| Artifact | What the reviewer MUST do | What it must NOT do |
|----------|---------------------------|---------------------|
| code diff | read the diff + run the tests + trace callers of changed symbols | propose architecture; review generated code |
| doc / brain note | verify every factual claim against a source; check the NOT-verified list exists | edit the note; add missing sections (that is a finding, not a fix) |
| config / JSONC | round-trip parse BOTH config roots; law-grep the 7 rules | review derived/runtime state |
| prompt / skill file | frontmatter parse; ASCII scan; gates-line definition present; no unicode-tags block chars | rewrite sections wholesale |
| deploy | walk the evidence chain (probe output, health check, rollback pointer) | accept "deployed" prose without a probe |
| merged crew artifact | run the END-TO-END acceptance commands on the composed artifact | accept per-fragment passes as coverage |

---

## APPENDIX D - THE ONE-QUESTION REVIEW (one-line diffs)

For a diff of <= 3 changed lines that touches no shared code, the full
packet collapses to ONE question + the caller trace:

```
review-class: one-line
question: "<does the changed value/expression still satisfy requirement Rn
           for every caller?>"
callers_checked: <list from git grep>
verdict: APPROVE | BLOCK
```

Everything else (rubric, coverage table) is waived for one-line diffs. What
is NEVER waived: the fresh session, the evidence rule, and the record.

---

## APPENDIX E - ADDITIONAL TROUBLESHOOTING ROWS

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Reviewer runs the build and edits a test to pass | reviewer had write access | read-only agent definition; write attempt = seat killed + logged |
| Findings reference files not in the diff | reviewer roaming beyond scope | out-of-diff findings require a caller-trace justification line or are dropped |
| Verdict JSON missing keys | reviewer ignored format | reject the review; re-run; format compliance is part of the verdict |
| BLOCK on a finding that is a preference ("I would name this differently") | ladder misread | it is Info; log reviewer for calibration |
| Review takes longer than the build | oversized packet | cap packet at the diff + 200 lines context; big features get per-join reviews instead of one mega-review |
| Two blocking rounds, then writer abandons | fix dispatch was too big | split: Criticals first, Majors second; two separate fix+re-verify rounds |
| Review record lost (run dir wiped) | non-durable run dir | write records under the repo's docs/runlogs/ or the session scratchpad, never only in chat |

---

## APPENDIX F - REVIEWER SEAT DEFINITION (agent file, drop-in)

```markdown
---
description: Fresh-context blocking reviewer (auto-review-gate seat)
mode: subagent
tools: [read, grep, glob, bash]   # NO write/edit
steps: 12
---

You are a blocking reviewer. You receive: a review packet (diff or artifact,
requirements, rubric). You never saw the work being done.

Procedure:
1. Read the artifact fully before any finding.
2. Run/execute what the packet allows; paste real output as evidence.
3. Restate each load-bearing claim as a question; try to refute it.
4. Findings: severity per ladder, evidence, one-line required action.
5. Requirements coverage table: met / unmet / unverifiable.
6. Verdict: BLOCK iff any Critical or Major. Output the JSON block.
Do not propose rewrites. Do not review style. If you cannot demonstrate a
defect with evidence, it is not a finding.
```

Pinning the reviewer as a custom agent definition (not an inline prompt)
makes the freshness structural: the agent definition carries no memory of
any build; every spawn is a new session by platform design.

---

## APPENDIX G - METRICS TO RECORD PER REVIEW

Append one line to the runlog per review (this is the calibration dataset
for judge calibration / D10):

```
review | <date> | artifact <path> | mode <phase> | verdict <APPROVE|BLOCK>
| findings c<C> m<M> i<I> | fabricated-later <count> | reopened-after-fix <count>
| rounds <n> | reviewer-model <id> | reviewer-session <ses_>
```

Consumed by: the promotion/demotion arithmetic (Section 5), honestscore
calibration (D10 judge calibration), and the quarterly "are our gates real"
audit. A review system that measures nothing cannot be promoted - and cannot
be trusted after it is.

---

## APPENDIX N - VERDICT JSON SCHEMA (the only machine-parsed output)

```json
{
  "verdict": "APPROVE",
  "mode": "blocking",
  "scope_statement": "correctness + R1..R3",
  "dimensions_walked": {
    "correctness": "checked",
    "completeness": "checked",
    "robustness": "checked",
    "security": "checked",
    "performance": "n/a: no speed claim and no hot path touched",
    "concurrency": "n/a: single-threaded handler, no shared state added",
    "docs": "checked"
  },
  "requirements_coverage": { "R1": "met", "R2": "met", "R3": "unverifiable" },
  "findings": [
    { "id": "F1", "severity": "Minor", "file": "src/x.py:12",
      "evidence": "<pasted output>",
      "required_action": "one line",
      "point_cost": 0.3,
      "closing_evidence": "<what output closes it>" }
  ],
  "unreadable": []
}
```

Rules: `verdict` is APPROVE | BLOCK. BLOCK requires >= 1 finding at
Critical or Major IN THE FINDINGS ARRAY (a BLOCK with no blocking finding
is invalid and re-run). "unverifiable" coverage rows do not block but MUST
appear in the final report's NOT-verified list. `unknown` on a walked
dimension is allowed with the reason; it caps honestscore, never hides.

## APPENDIX N2 - SKIP CONDITIONS, EXACTLY

A review is skipped only for:

1. `skipped: no diff this turn` - no file/artifact changed in the turn
   (read-only research, pure Q&A). The NEXT diff re-fires the gate.
2. `skipped: trivial ask` - conversational turn, no artifact exists.
3. `skipped: no spawn tool - self-run` - the degraded path above.

Anything else ("time pressure", "small change", "already tested") is an
UNAUTHORIZED skip: the done-claim proceeds, and the audit counts it as a
gate bypass. The once-per-rollout V3 test (Section 6) exists precisely to
catch these bypasses early.

---

## APPENDIX K - PER-DIMENSION REVIEW CHECKLISTS (aligned to honestscore)

The reviewer walks the applicable dimensions; each carries concrete
questions. Mark each `checked | n/a (+why) | unknown` in the verdict JSON.
"n/a" for a dimension the artifact plausibly exercises is forbidden -
honestscore's rule applies here too. An "unknown" caps that dimension per
the honestscore Unknown rule (<= 6.0), so prefer checked-with-evidence.

### K.1 correctness
- Does every changed path do what the requirement LITERALLY says (not what
  the implementation habitually says)?
- Were the acceptance commands actually run, or only claimed? (Read the
  runlog; a claimed run with no pasted output is a Major finding.)
- Was any test weakened, skipped, or deleted in this diff (Critical if yes)?
- Do changed shared symbols break callers (trace the regression cone with
  grep before answering)?

### K.2 completeness
- R1..Rn coverage table: met / unmet / unverifiable per requirement.
- Any requirement with no owner fragment (unassigned remainder)?
- Does the diff respect the stated out-of-scope boundary (out-of-scope
  means untouched, not half-touched)?

### K.3 robustness / edge cases
- Empty / huge / unicode / concurrent inputs on the new code paths.
- Failure paths: what happens when the dependency is down, slow, or lying?
- Timeouts on every external call (network, subprocess, DB)?

### K.4 security
- New input at a boundary -> validated / parameterized?
- Any secret echoed, logged, or committed (zero tolerance, Critical)?
- Auth check on every NEW route/handler (the one-path-skip defect class)?
- Injection sinks: string-built SQL/HTML/commands on new input paths?

### K.5 performance
- Any speed claim in the diff? If yes: where is the measurement?
- O(n^2) over a hot collection; N+1 query pattern; per-item HTTP calls
  that could batch.
- Evidence tiers: before/after with output -> full credit; a single timing
  run or profile screenshot -> <=8.0; "should be faster" -> Info only.

### K.6 concurrency
- Shared mutable state touched? Locks, atomics, or a documented
  single-writer guarantee?
- Race tooling run where the language has it (go test -race; Python
  xdist footguns; TS unawaited promises in new code)?
- The worktree / single-writer rules respected for parallel agents?

### K.7 docs
- Public behavior changed without doc/CHANGELOG touch (per repo convention)?
- New config / env vars documented?
- Runbook updated for the failure modes this diff introduces?

The checklist is also the reviewer's memory aid under step limits: a seat
capped at 12 steps can still walk all 7 dimensions with one pass each.

---

## APPENDIX L - WORKED DIALOGUE (what a good review exchange looks like)

Implementer -> reviewer packet:

```
artifact: diff 4f2c1a..9d7e02 (2 files, +38 -12): rate limiter key change
requirements:
  R1: per-tenant limits (was per-IP)
  R2: no limit change for tenants below 100 req/min
  out of scope: the dashboard, the admin API
rubric: auto-review-gate Sections 2.3 + Appendix K
```

Reviewer (fresh session) findings:

```
F1 | Critical | src/limiter.py:88
evidence: tenant key falls back to "anon" when the header is missing;
  every header-less request shares ONE bucket (grep: no other key source).
required_action: reject unauthenticated requests before limiter lookup, or
  hash the socket peer into the fallback.
F2 | Major | src/limiter.py:41
evidence: git grep "req/min" shows the dashboard still reads the old
  per-IP counter - it will render zeros for all tenants.
required_action: point the dashboard read at the new counter, or out-of-
  scope the dashboard in the spec BEFORE merge (pick one, in writing).
F3 | Info | README not updated; R1 changes operator-visible behavior.
```

Verdict: BLOCK (1 Critical). One fix dispatch. Re-review (fresh session)
confirms F1 by running the unauth-probe command, F2 resolved by the written
boundary note. APPROVE. Total cost: two small spawns. The Critical that
would have anonymous-ratelimited the whole tenant pool behind one bucket
never shipped.

---

## APPENDIX M - FAQ (short, factual)

**Q: Does the reviewer review the implementer's honestscore?**
No. The reviewer sees artifact + requirements + rubric. The MIN is computed
after; where both exist, the LOWER number is final (honestscore rule 7).

**Q: Can one reviewer session review two unrelated diffs?**
Yes - freshness is per-BUILD, not per-file. What voids it is having helped
build the diff or having seen the implementer's reasoning.

**Q: No spawn tool on this runtime?**
Degraded path: re-read the artifact in a NEW turn with the scope line and
the Step 0b question-form; write the one-question review for small diffs;
report `review: self-run (no spawn tool)`. honestscore still caps per its
Unknown rule - say so. Honest degraded review beats skipped review.

**Q: Do hotfixes need the full procedure?**
The packet collapses (one-question review, Appendix D). The fresh session
and the written record never collapse. Blocking mode stays on.

**Q: Who reviews the reviewer?**
The fabricated-later metric and the demotion arithmetic (Section 5). A
reviewer is trusted at exactly the rate its findings survive adjudication.

**Q: Defect found in UNRELATED code seen while reading?**
Reachable from the diff (same call path) = in scope. Untouched code = log
as Info with a pointer; do not block, do not fix from the reviewer seat.

---

## APPENDIX P - REVIEW OF SKILL / PROMPT FILES (fleet self-review)

Skills and prompts are artifacts; they get the same gate with their own
checklist:

- YAML frontmatter parses (name / description / alwaysFire present).
- ASCII-safe: no typographic quotes, em-dashes, arrows, or Unicode Tags
  block characters (U+E0000-U+E007F crash some runtimes). Mechanical scan:

  ```
  python -c "import sys;s=open(sys.argv[1],encoding='utf-8').read();bad=[(i,c) for i,c in enumerate(s) if ord(c)>126];print(len(bad),'non-ascii chars');[print(hex(ord(c))) for _,c in bad[:10]]" <file>
  ```

  Expect `0 non-ascii chars`.
- The gates-line definition present and matching the canonical-14 spec.
- Line count within the 800-3,000 install bar.
- No secrets; no machine-specific absolute paths stated as universal;
  every version/price/CVE claim either web-verified (fetch date noted) or
  marked unverified - a confident unsourced number is Major.
- Worked examples walked through the procedure by the reviewer (an example
  that contradicts the procedure is a Major - examples teach the next
  session, and a wrong example is worse than none).

Skill-file reviews are the cheapest blocking reviews in the fleet (single
file, no build) and they run at full blocking from day one.
