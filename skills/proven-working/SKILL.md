---
name: proven-working
description: Multi-check verification gate - prove every claim with 6-8 EXTERNAL checks before calling work done. Build trace, terminal test with real output, independent re-read by a different mechanism, adversarial refutation pass, regression trace, docs check, edge cases, security pass. Re-reading your own answer is not verification - a check must have been able to come out the other way. Use before claiming done/fixed/shipped, for high-stakes changes, and whenever a completion claim needs evidence.
alwaysFire: true
---

# proven-working - The Multi-Check Verification Gate

## SECTION 0 - PURPOSE AND SCOPE

This skill is the final verification gate applied to high-stakes changes before
any done/fixed/shipped claim. It is deliberately distinct from its twin-domain
neighbor:

| Skill | Question it answers | When it runs |
|---|---|---|
| deep-test-all | "Is the code correct, and are the tests real?" | While building and debugging |
| proven-working | "Has every CLAIM been externally checked before emission?" | The final gate before the claim |

deep-test-all produces test evidence. proven-working audits the CLAIMS with
external checks that could have come out the other way. Re-reading your own
answer is not verification: the same reasoning that produced the error
re-confirms it. What catches errors is a check that could have come out the
other way - a command with an exit code, a different machine reading the
result back, an adversarial pass told to refute, a test whose failing branch
you have actually seen fail.

The load-bearing rule, measured repeatedly in real sessions: every "done" that
was wrong was caught by an EXTERNAL check, never by another read-through.
Target 6-8 such checks on anything high-stakes. If you cannot name what each
one checked, you have not done them.

Out of scope: writing the tests (deep-test-all), judging prose quality
(honestscore), hashing the evidence bundle (skill-paperclip-ledger-guard).

## WHEN THIS GATE FIRES

- Any turn that ends in a done-claim, fixed-claim, shipped-claim, or
  deployed-claim.
- Any high-stakes change: migrations, auth/security, money, deletes, shared
  code, deploys, anything hard to reverse.
- Whenever the user says "go deep", "proven working", "100% mode", or flags
  the change as release-blocking.
- In multi-agent runs: before a seat's final JSON is emitted, and at the
  orchestrator's merge point across seats.

Skipped (honestly reported) on: conversational turns, trivial lookups, and
mid-task progress beats with no completion claim.

## THE EIGHT CHECKS (run 6-8; all 8 for critical code)

A check qualifies ONLY if it could have come out the other way. A check that
cannot fail is decoration.

### C1 - Build trace

- State the exact build command with flags and target.
- Paste real output from THIS session, warnings included.
- Exit code shown unpiped (PowerShell `$LASTEXITCODE`; bash `echo $?`
  immediately after the command).
- Note incremental builds; release gates build clean.
- Known trap: "build succeeded" while tests never ran. C1 proves compilation;
  only C2 proves behavior.

### C2 - Terminal test (red before green)

1. Name the test that encodes the requirement.
2. Watch it FAIL once; the failure message must name the missing behavior,
   not a setup error.
3. Apply the change; watch the same test go green.
4. Paste both observations.

No existing test: write it first (red), then the code. Pure refactor: capture
green BEFORE and AFTER - "suite unchanged" without a fresh run is the
stale-green trap.

### C3 - Independent read-back

| Mechanism | Command shape | Proves |
|---|---|---|
| Remote state read-back | `git ls-remote origin <branch>` | push landed server-side |
| Public endpoint probe | `curl -s https://<live-url>/health` | deploy actually serving |
| Second-process hash | `sha256sum file` (other machine/session) | content integrity |
| Filesystem stat | `Get-Item` / `ls -la` (size + mtime) | artifact exists, fresh |
| DB read-back | SELECT the written row | persistence happened |
| API read-back | GET the created resource | remote state matches claim |

Rules:

- The mechanism must differ from the one that PRODUCED the artifact.
- Deploys read back through the PUBLIC path the user hits, not the internal one.
- The read-back must show THIS session's change (a distinguishing value),
  not merely "it exists".

### C4 - Adversarial refutation pass

1. Frame: "Assume this change is wrong. Show the input that breaks it."
2. Feed the pass ONLY the diff and the requirement - not the author's
   reasoning - so it cannot inherit the same blind spot.
3. Ask for refutations, not ratings.
4. Triage every refutation: real defect (fix), spec gap (escalate), or
   invalid (one-line rebuttal naming the evidence).
5. A clean bill must still name what it checked; "found nothing" without
   scope means it did not look.

Scope the pass to correctness + the stated requirements ONLY. An unscoped
"find problems" pass invents defects; a pass told to refute must be able to
disagree.

### C5 - Regression trace

- Rename/signature change: grep every call site; open each; verify or update.
- Shared-code behavior change: list every consumer; write a one-line verdict
  per consumer on whether the new behavior breaks its contract.
- Config/flag change: enumerate every environment where the flag differs.
- Dependency bump: changelog for breaking changes; lockfile diff; full suite.
- Shared helper edit: run the tests of EVERY importer, not just this package.

### C6 - Docs and claim ledger

- Every number: value + source file + method. No method = not evidence.
- Every "verified": the command that would have caught the failure.
- README/CHANGELOG/comments updated in the same change; orphaned claims about
  old behavior corrected.
- No doc states a number the repo cannot recompute.
- Inherited evidence (another session's paste) labeled INHERITED; unverified
  until re-run locally.

### C7 - Edge cases at real boundaries

Minimum probe set, mapped to the change:

```
empty input | null/None | zero | negative | max boundary +1
unicode (combining marks, emoji, CJK) | CRLF/newline | very long string
double-submit / replay | expired token / clock skew | concurrent duplicate
wrong tenant/owner id | binary-in-text | trailing whitespace
```

Run the probes; record actual outputs. "Would obviously pass" is not a run.

### C8 - Security pass

Trigger: any change touching auth, sessions, input parsing, file paths, SQL,
secrets, or outbound calls.

- Input validated at the boundary; parameterized SQL; no eval/exec on input.
- Expired token rejected; cross-tenant access denied; escalation path traced.
- No secrets in code, logs, or error messages.
- Session fixation, rotation on privilege change, logout invalidation checked.
- XSS sinks escaped; CSP intact; redirects validated.

## THE CLAIM LEDGER

Every high-stakes claim gets a ledger row before it is emitted:

```
CLAIM LEDGER
| # | Claim                        | Check | Result                          |
|---|------------------------------|-------|---------------------------------|
| 1 | Migration script builds      | C1    | exit 0, output pasted           |
| 2 | Backfill: 10k rows, 0 missed | C2    | red first (old query), green    |
| 3 | Counts match on read-back    | C3    | 10,000 = 10,000, second conn    |
| 4 | Adversary probes rejected    | C4    | 3 probes, clean rejects         |
| 5 | Old-column callers compile   | C5    | 4 call sites, 4 pass            |
| 6 | "10,000" recomputable        | C6    | SELECT COUNT matches claim      |
| 7 | Unicode/max-length probed    | C7    | round-trip stored               |
| 8 | No secrets in logs           | C8    | log scan clean                  |
```

- 6-8 rows for high-stakes changes (all 8 for critical code).
- A row without a named check reads UNVERIFIED, and the verdict is UNTESTED.
- Every row names its command so any other session can re-run it. Evidence
  that cannot be re-run is testimony, not evidence.

## WHAT DOES NOT COUNT AS A CHECK

- Re-reading your own answer 6-8 times. The same reasoning that produced the
  error produces the same review.
- "It compiled" without a test that executed.
- A screenshot of the first viewport only. Judge at real size, full scroll.
- A proxy metric without opening the artifact. Laplacian variance ranked the
  grittiest sharpening best; edge energy ranked the least-legible icon best.
- A metric passing a rotated shape at 0.96. Outline math is not perception.
- A metric that cannot fail. If it could not have come out the other way,
  it is not a check.
- A process list match (pgrep matches its own command line). Side-effects,
  not process names.
- A number with no method. A fabricated "47.1 RMS" reached a report once and
  justified a code change it did not support.

## PROXY METRIC RULES (from measured failure, never skipped)

1. A proxy metric may rank variants of the SAME treatment; it may never
   choose BETWEEN treatments. To pick between treatments, open the artifact
   and judge the thing the user sees, at the size the user sees it.
2. A guard does not exist until PROVEN TO FAIL on the defect it was written
   for. Re-introduce the original defect and watch the guard exit non-zero.
3. Every number written down must be recomputable from files that exist, with
   the method stated.
4. Judge at the size and context the user actually gets - not at 3x zoom, not
   off-canvas, not on a downscaled thumbnail.
5. "Could not measure" is NEVER a pass. Score it: 0 pass / 1 defect /
   N unmeasured, and say which.

Meta-rule: when one of these is violated, the fix is a CHECK, not a resolution
to be more careful. Knowing rule 1 did not stop rule 1 being violated again
the same day.

## THE UNTESTED PROTOCOL

When a check cannot run (missing credential, no environment, vendor down):

```
UNTESTED: <claim>
  blocked by: <specific missing input>
  exact command to verify when available:
    $ <command>
```

- The task is NOT done. Status stays open. No DONE in any status field.
- An honest "untested" with the exact command beats a confident guess.
- "Probably works", "should be fine", and "I recall that" are all the same
  failure: reasoning where a command was available.

## DELIVERY FORMAT

High-stakes deliveries follow this order:

```
1. What changed (files + one line each)
2. Claim ledger (the table above)
3. Verification evidence (commands + real output, unpiped exit codes)
4. Regression trace (callers + per-caller verdict)
5. Open items (each UNTESTED with its verify command)
```

"Done" without rows 2-3 is a blocked claim, not a delivery. Sections 4-5 may
be empty; they may not be missing.

## SHORT GATE (small, low-risk changes only)

A one-line, low-blast-radius fix needs the SPIRIT of the gate, not all of it:

1. The one command that could fail on this change (C2 compressed).
2. The read-back of the artifact through a different mechanism (C3).
3. The claim table with a source for every number (C6).

Three rows, pasted. A one-line diff never needs the full ceremony - but it
never gets zero checks either.

## MULTI-AGENT CONTEXT

- Each seat's final JSON carries an `evidence` block per claim. A status of
  "done" without evidence is treated as "unverified" by the orchestrator.
- Seats never inherit another seat's green without re-running one check.
- The orchestrator re-runs one check per merged seat (the cheapest C3-class
  read-back) - self-reported done is a claim, not evidence.
- Cross-seat changes get a C5 trace across the boundary: the consumer seat
  pins the contract with a test, not a promise.

## ANTI-PATTERNS (each one shipped a real false-done)

1. The stale green - "tests passed" from a run before the last edit.
2. The unfalsifiable claim - "it passes" where the check has no failing branch.
3. The invented metric - a number quoted with no method or source.
4. The judged thumbnail - a visual verdict read off a downscaled render.
5. The off-canvas measurement - CSS overflow counted from invisible elements.
6. The exit code through a pipe - a gate piped into tail returned 0 on reject.
7. The unfalsifiable claim - a check with no failing branch.
8. The praised review - an "adversarial" pass that found only compliments.

## INTEGRATION WITH THE GATE LINE

The done-turn gate line carries this skill's status:

```
gates - ... | proven-working: fired 7/8 (C1,C2,C3,C4,C5,C6,C7) | ...
```

- `fired N/8` - checks run and pasted this turn.
- `skipped: why` - only on turns with no done-claim, or SHORT-gate turns
  naming the subset.
- An honest "skipped: why" is acceptable; a silent omission is a violation.

## THE EIGHT CHECKS - WORKED DETAIL

### C1 in depth

- The command is stated exactly as typed, including flags and target.
- Output pasted from THIS session, warnings included - a cleaned paste is
  testimony, not evidence.
- Exit code unpiped: PowerShell `$LASTEXITCODE`; bash `echo $?` immediately
  after the command, before anything else runs.
- Incremental builds noted as such; release gates build clean from a fresh
  checkout.
- Build success is C1. Test EXECUTION is C2. A green C1 with no C2 proves
  compilation and nothing else.

### C2 in depth

1. Name the test that encodes the requirement.
2. Watch it fail once; the failure must name the missing behavior, not a
   setup error. A test failing for the wrong reason has shown nothing.
3. Apply the change; watch the same test go green.
4. Paste both observations in the ledger row.

No existing test: write it first (red), then the code. Pure refactor: fresh
green before AND after - "the suite is unchanged" without a run is the
stale-green trap.

### C3 in depth

The mechanism must differ from the producer. Writing a file and reading it
with the same in-memory handle is not read-back. Deploys read back through
the PUBLIC path, not the internal one - internal success with a broken
public path is a classic false-done. The read-back must show a distinguishing
value from THIS change (new field, new timestamp, new count), not just
existence.

### C4 in depth

The pass receives ONLY the diff + requirement - not the author's reasoning -
so it cannot inherit the same blind spot. It must return refutations
("show an input where behavior is wrong"), not ratings. Every refutation is
triaged: real defect (fix), spec gap (escalate), invalid (one-line rebuttal
with evidence). A clean bill still names what it checked; "found nothing"
without scope means it did not look.

### C5 in depth

- Rename/signature: grep every call site; open each; update or verify.
- Behavior change in shared code: per-consumer contract verdict, one line each.
- Config/flag: enumerate every environment where the value differs.
- Dependency bump: changelog + lockfile diff + full suite.
- Shared helper: run EVERY importer's tests.

### C6 in depth

- Every number: value + source + method.
- Every "verified": the command that would have caught failure.
- Every status-table "done": an evidence pointer, or the row reads UNTESTED.
- Docs corrected in the same change, never left asserting the old behavior.

### C7 in depth

Pick probes that map to the actual change; run them; paste actual outputs.
The probe set in the main section is the minimum; concurrency probes
(double-submit, parallel duplicate) are mandatory for shared state.

### C8 in depth

- Boundary validation; parameterized SQL; no eval/exec on input.
- AuthN/AuthZ: expired rejected, cross-tenant denied, escalation traced.
- Secrets: env/vault only; none in code, logs, or errors.
- Session: fixation, rotation on privilege change, logout invalidation.
- Output: escaping, CSP, redirect validation.

## WORKED EXAMPLE - A HIGH-STAKES FIX THROUGH THE GATE

Claim: "Migrated the users table to the new email format; backfill complete."

```
| # | Claim                                   | Check | Result                          |
|---|-----------------------------------------|-------|---------------------------------|
| 1 | Migration script builds                 | C1    | exit 0, output pasted           |
| 2 | Backfill test: 10k rows, 0 missed       | C2    | red first (old query), green    |
| 3 | Row counts match on replica read-back   | C3    | 10,000 = 10,000 via second conn |
| 4 | Adversary: duplicate emails, NULLs      | C4    | 3 probes, all rejected cleanly  |
| 5 | Callers of old column still compile     | C5    | 4 call sites greped, 4 pass     |
| 6 | "10,000 rows" is recomputable           | C6    | SELECT COUNT matches claim      |
| 7 | Unicode + max-length emails probed      | C7    | 2 probes, stored round-trip     |
| 8 | No credentials in migration logs        | C8    | log scan clean                  |
```

Verdict: PROVEN (8/8). The bundle (deliverable hash + evidence hash) then
goes through the claim gates. If C2 was never observed red, row 2 reads
"unverified - no failing observation" and the verdict is UNTESTED regardless
of prose confidence.

## CONTRADICTION HANDLING PROTOCOL

When two checks disagree (suite green, endpoint 500; hash matches, size differs):

1. STOP emitting. A contradiction is a finding, not noise.
2. Name the pair and the delta in one sentence.
3. Re-run BOTH fresh - stale evidence is the most common cause.
4. Still contradictory: the lower-level check wins provisionally; the
   discrepancy is investigated before any done-claim.
5. Log the contradiction; never average it away.

## DOCS CHECK (C6 detail)

- README/CHANGELOG reflect the new behavior.
- Comments asserting the OLD behavior are corrected, not orphaned.
- Migration notes state what the deployer runs, in order.
- No doc states a number the repo cannot recompute.

## RIGOR SCALING TABLE (match ceremony to blast radius)

| Change type | Gate | Checks |
|---|---|---|
| Comment/typo/local rename | none | read the diff once |
| One-line internal fix, tests exist | SHORT | C1, C2, C5 |
| New function / behavior change | SHORT+ | C1, C2, C5, C7 |
| Shared code / public API | FULL | C1-C7 |
| Deploy / auth / money / delete | FULL | C1-C8, all pasted |

Ceremony scales with blast radius; the paste is never optional.

## FAILURE SIGNATURE CATALOG (how each false-done actually looked)

| Signature | What it looked like | The check that catches it |
|---|---|---|
| Piped exit code | gate printed FAIL; pipeline exited 0 | exit code unpiped |
| Stale green | "tests pass" from before the last edit | fresh run after final edit |
| Fold-only screenshot | overflow below the fold invisible | full-page, real size |
| Off-canvas count | defect counted from display:none elements | measure the visible DOM |
| Proxy verdict | sharpest-looking output was least legible | open the artifact, judge at display size |
| Fabricated metric | "47.1 RMS" reproducible by no method | every number recomputable |
| Deleted test | suite green, coverage silently dropped | never weaken a test to get green |
| Self-matching process check | pgrep matched its own command line | verify side-effects, not names |

## EVIDENCE PERSISTENCE (so the checks survive the session)

For multi-session, overnight, or multi-seat work, the ledger goes to disk:

```
<run-dir>/evidence/
  C1-build.txt        command + output + exit code
  C2-red.txt          the failing observation
  C2-green.txt        the passing run, same command
  C3-readback.txt     independent mechanism output
  C4-adversary.md     refutations + triage
  C5-regression.md    call-site list + verdicts
  C6-ledger.md        the claim table
  C7-edges.txt        probes + actual outputs
  C8-security.md     results + findings
```

- First line of each file names the claim it evidences.
- Quotes of other sessions' output are labeled INHERITED.
- The claim-gate bundle hashes these files; a missing file fails the bundle.

## SELF-SCORING BEFORE EMISSION (bounded loop)

Before the done-claim, score the delivery on the honest rubric
(see honestscore) and loop on logged defects:

- Loop until >= 9.0 (sites/brain-docs) or >= 9.5 (critical code).
- Stop early only at 100% honest certainty that a threshold is unreachable,
  naming the blocker and the missing input.
- A pass with unmeasured dimensions is 0 pass / N unmeasured - never a pass.

## DEPLOY-SPECIFIC ADDENDUM (when the change ships)

1. Read back the PUBLIC path (`curl https://<live-url>/...`) and diff against
   expectation - internal health endpoints lie.
2. The response must carry a distinguishing value from THIS deploy (build id,
   new field, changed copy). "It responds" is not evidence of the new version.
3. Rollback is verified BEFORE shipping: previous tag noted, one-command
   rollback tested or stated as untested.
4. Post-deploy window: one error-rate/health check after the traffic warms.

## EVIDENCE LIFETIME AND STALENESS

- Any check that ran BEFORE the last edit is void; every check re-runs after
  the final edit, or it describes a different artifact.
- Time-sensitive facts (versions, prices, limits, CVEs) come from an
  authoritative current source in-session, or they are flagged unverified.
- Inherited green (another session's paste) is unverified until re-run
  locally by the session making the claim.

## EVIDENCE FILE CONVENTIONS

For multi-session or multi-seat runs:

- Each check's raw output lands in `<run-dir>/evidence/C<n>-*.txt`.
- First line of every file names the claim it evidences.
- Quotes of other sessions' output are labeled INHERITED and unverified until
  re-run locally.
- The bundle hashes these files; a missing file fails the bundle.

## WORKED EXAMPLE - DOCS/REPORT DELIVERABLE THROUGH THE GATE

Claim: "Report written, all sections complete, 9.5/10 quality."

```
| # | Claim                              | Check | Result                          |
|---|------------------------------------|-------|---------------------------------|
| 1 | File renders, no encoding breaks   | C1    | ASCII scan: 0 non-ASCII lines   |
| 2 | All required sections present      | C2    | grep each header: 11/11 found   |
| 3 | Line/byte count via independent wc | C3    | 1,204 lines, pasted             |
| 4 | Adversary: slop/padding/overclaim  | C4    | 2 refutations, both fixed       |
| 5 | References resolve                 | C5    | every linked file exists        |
| 6 | Score recomputable from rubric     | C6    | rubric + per-dimension scores   |
| 7 | Judged at display size             | C7    | previewed at real size          |
```

"It looks good" is not a row. Judgment is delegated to a scoped rubric;
everything else is a command with output.

## CHECK COUNTER (quick reference)

| Check | One-line purpose | Void when |
|---|---|---|
| C1 build | artifact builds | output summarized, not pasted |
| C2 terminal test | behavior red then green | never observed failing |
| C3 read-back | independent mechanism confirms | same mechanism as producer |
| C4 adversarial | refutation attempted, triaged | pass praised instead of refuting |
| C5 regression | callers traced | only the touched file examined |
| C6 claims | every number recomputable | metric without method |
| C7 edges | boundaries probed for real | "would obviously pass" |
| C8 security | auth/input/secrets traced | skipped because "not security" |

## DECISION QUICK TABLE

| Situation | Gate action |
|---|---|
| Done-claim, zero pasted outputs | BLOCK; demand evidence |
| Only re-reads as evidence | BLOCK; require one runnable external check |
| Evidence predates the last edit | Re-run; claim unverified |
| Adversarial refutation returned | Fix or rebut with evidence; never argue past it |
| Check impossible right now | UNTESTED with the exact verify command |
| Contradictory checks | Stop; stale-first re-run; then escalate |
| Subagent self-reports done | Orchestrator re-runs one check of its evidence |
| Number without source | Recompute from a file or delete the number |

## EMERGENCY MINIMUM (when time-boxed, in priority order)

1. The ONE command that could fail on this exact change.
2. The read-back through a different mechanism.
3. The claim table with a source for every number.

A done-claim with fewer than these three is not a claim - it is a hope.

## FAQ

**Q: Is 8 checks overkill for a one-line fix?**
Yes - use the SHORT gate (3 checks). The 6-8 rule is for high-stakes changes;
match ceremony to blast radius, never skip the paste.

**Q: What if two checks contradict?**
That is the system working. Investigate before emitting; the contradiction is
the defect being handed to you.

**Q: The change is tiny and I "obviously" know it works.**
The obvious ones are the ones that were wrong. A 20-second command beats a
20-minute revert.

**Q: Who verifies the verifier?**
Every ledger row names its command, so any other session can re-run it.
Evidence that cannot be re-run is testimony, not evidence.

**Q: The adversarial pass keeps praising the work.**
Re-scope it: correctness only, refutations only, and require it to name what
it checked. A pass that cannot disagree must still list what it examined.

## APPENDIX F - RUNBOOK: THE GATE UNDER PRESSURE

Real sessions get squeezed. The gate survives compression in a fixed order;
what it does not survive is improvisation.

When the turn is long and the claim is near:

1. Freeze the diff. No edits after the first check runs - every check after
   the last edit, or the whole ledger is void.
2. Run the emergency minimum (C2-compressed, C3, C6). Three rows minimum.
3. If anything fails, the verdict is UNTESTED - not "mostly done".
4. If a credential/environment is missing, name it in one line and stop.
   The missing input is the user's to supply; everything else is done.
5. Emit the gate line with `fired N/8` and the list of what each check covered.

Pressure phrases that predict a false-done - treat each as a trigger to slow
down, not speed up:

- "just this once" / "one-line change" / "obviously fine"
- "I already ran it earlier" (when the last edit came after)
- "the user is waiting" (a wrong answer costs more than a slow one)
- "it worked yesterday" (stale green)

## APPENDIX G - SCORING THE GATE ITSELF

Meta-checks run quarterly or after any incident:

1. Introduce a real defect; confirm the gate blocks the done-claim.
2. Confirm a piped-exit-code error is caught (the gate checks codes unpiped).
3. Confirm the ledger rejects a row whose evidence predates the last edit.
4. Confirm an adversarial pass that "praises" is rejected for missing scope.

Each meta-check records: date, defect injected, gate behavior observed. Three
consecutive clean audits retire nothing; one missed defect appends an incident
to Appendix A and a probe to C7.

## APPENDIX H - THE TEN-STRIPE LEDGER (compact form for the gate line)

For turns where the full table is too heavy, the ledger compresses to one
line; the rules are identical, only the form changes:

```
pw: C1:0|C2:red>green|C3:sha-match|C4:2-refutations-fixed|C5:4-sites-ok|C6:recomputed|C7:5-probes|C8:n/a
```

Reading rules:

- Every check shows its outcome, never its intention. `C2:red>green` means
  both observations exist and are pasted above the line.
- `skipped` is always `skipped:<reason>`. An unnamed skip is a violation.
- A numeric claim in the message body with no matching C6 row is a defect in
  the response, whether or not the code is correct.

## APPENDIX I - HANDOFF TO THE CLAIM GATES

proven-working produces the evidence; the claim gates consume it. The handoff
contract, so the gates never have to re-derive anything:

1. Gate 2 (skill-anti-overclaim-upgrade) scans the RESPONSE TEXT for blocked
   phrases in done-claim context. proven-working's verdict replaces prose:
   "PROVEN (7/8)" instead of "the migration is complete".
2. Gate 3 (skill-paperclip-ledger-guard) hashes the deliverable content and
   the pasted evidence output. The ledger rows ARE the evidence text; paste
   them before hashing.
3. Gate 4 (anti-overclaim base) requires the completion-evidence block. The
   ledger satisfies it verbatim when the rows are pasted.

Failure at any gate returns the work to this skill with the block reason; the
fix is always the same shape: run the missing check, paste the real output,
re-emit the verdict.

## APPENDIX J - TWIN BOUNDARY (proven-working vs deep-test-all)

The two skills share a subject (verification) and are often fired together.
The boundary, stated once so neither drifts into the other:

| | deep-test-all | proven-working |
|---|---|---|
| Time of fire | during the work | before the claim |
| Question asked | "does the code behave correctly?" | "is every CLAIM externally checked?" |
| Unit of work | functions, branches, cases | statements, numbers, verdicts |
| Output | test evidence (case table, red/green, mutation log) | claim ledger (6-8 rows) |
| Failure mode it prevents | silent defects | false dones |
| Unique rule | fail-power: a test proves nothing until seen failing | independence: a check must be able to come out the other way |

Overlap is allowed and normal: both pasted outputs, both demand this-session
freshness, both refuse summarized evidence. What they never share is the
moment of firing - tests are built during the work; claims are audited at
the end. A session that runs only one of the two half-finished its job.

## APPENDIX K - FIELD NOTES (verbatim patterns that failed)

Collected from real sessions; each is the compressed shape of a false-done
that shipped, kept here because prose descriptions fade and patterns are
recognizable at a glance:

```
"Done."                                    -> no ledger, no evidence, blocked
"Migration complete and tested."           -> "tested" with no runner output
"Deployed to production successfully."     -> no public-path read-back
"All 4 call sites updated."                -> count asserted, grep never run
"Works on my machine."                     -> not a check on any machine
"Score: 9.5/10."                           -> scorer and rubric not named
"Exit 0 (verified)."                       -> code read through a pipe
"Screenshot confirms layout."              -> first viewport only, unscrolled
"Tests pass." (last edit after last run)   -> stale green
"Coverage is 100%." (line mode only)       -> branches untested
"The API returns the correct JSON."        -> which command showed it?
"No secrets in the logs."                  -> log scan output not pasted
```

Each entry maps to a check: C2, C3, C5, C6, C7, or C8. When a new false-done
shape appears in the wild, it gets one line here and one probe in C7.

## APPENDIX L - WHY EXTERNAL, NOT INTROSPECTION

The owner's rule this skill implements, and the measurement behind it:
re-reading your own answer six to eight times mostly re-confirms it, because
the same reasoning that produced the error produces the same review. What
catches errors is a check that could have come out the other way:

- a command with an exit code
- a different machine reading the result back
- an adversarial pass told to refute it
- a test whose failing branch you have actually seen fail

Target 6-8 such checks on anything high-stakes, and if you cannot name what
each one checked, you have not done them. Measured repeatedly in the fleet:
every wrong "done" was caught by an external check; zero were caught by
another read-through. That asymmetry is the whole reason this skill exists.

A useful review comes from fresh context, scoped to correctness and the
stated requirements. A reviewer told to "find problems" invents them and
drives over-engineering; a reviewer told to REFUTE the claim with a named
input finds the real ones.

## EXIT CHECKLIST

- [ ] C1 build trace: command + output + exit code, this session
- [ ] C2 terminal test: key test shown red at least once, then green
- [ ] C3 independent read-back by a different mechanism
- [ ] C4 adversarial refutation pass run and answered
- [ ] C5 regression trace over changed symbols' callers
- [ ] C6 claim ledger: every stated number/name recomputable
- [ ] C7 edge cases at real boundaries probed
- [ ] C8 security pass for auth/input/secrets changes
- [ ] 6-8 rows in the ledger (or SHORT gate rows for small changes)
- [ ] Every skip explicit; verdict PROVEN or UNTESTED
- [ ] Evidence bundle handed to the claim gates (Gate 2 -> 3 -> 4)

## APPENDIX A - INCIDENT FILE (the false-dones this gate exists for)

Each entry: what was claimed, what was actually true, the check that would
have caught it. These are the reasons each check exists; read them when
tempted to skip a row.

1. **The piped FAIL.** A push-gate printed REJECT; the pipeline read the exit
   code of `tail` instead of the gate. Ship proceeded on a rejected action.
   Fix: exit codes checked unpiped, immediately, in the emitting session.

2. **The off-canvas defect.** Overflow counted from elements outside the
   visible area; the count said 0 defects; users saw the scrollbar. Fix:
   measure the visible viewport at the size the user gets.

3. **The unscrolled screenshot.** Page declared fixed from a first-viewport
   capture; the break was two scrolls down. Check: full-scroll capture or a
   DOM measurement of the rendered box.

4. **The thumbnail verdict.** An icon judged "clean" at 3x zoom shipped grit
   at the 110px it actually displays at. Judge at display size and context.

5. **The rotated disc.** Silhouette IoU 0.96 for a mark rotated 35 degrees.
   Outline metrics cannot judge artwork; use perceptual or adversarial checks.

6. **The fabricated RMS.** A number reached the brain, justified a change,
   and reproduced by no method. The real value reversed the conclusion.
   Rule: every number is recomputable from files that exist, method stated.

7. **The deleted test.** Suite went green because a test was removed, not
   because the bug was fixed. Rule: never weaken a test to force green; a
   legit test fix is argued openly in the report.

8. **The self-matching process check.** pgrep matched its own command line;
   "daemon running" was self-confirmation. Verify side-effects: a port
   answering, a file growing, a log line with today's timestamp.

9. **The sibling-seat inheritance.** Seat B marked its task done because the
   upstream seat "said done" with no evidence block. Treated as UNVERIFIED;
   re-verified locally; upstream was in fact broken.

10. **The internal-health deploy.** Internal /health returned 200; the public
    URL served the previous build. Read back the path the user actually hits.

11. **The stale-green commit.** Verification output quoted from a run before
    the last edit. Every check re-runs after the final edit or it is void.

12. **The unfalsifiable pass.** "The validation works" - the test asserted
    `result is not None`. Rewrote to assert the actual expected value; it
    immediately caught the defect. Tests that cannot fail are decoration.

## APPENDIX B - THE SHORT GATE, WORKED

Change: a one-line fix to a date formatter used in two places.

```
| # | Claim                        | Check | Result                              |
|---|------------------------------|-------|-------------------------------------|
| 1 | Project still builds         | C1    | exit 0, output pasted               |
| 2 | New case: "2026-02-29" path  | C2    | red (old code), green (fixed)       |
| 3 | Callers traced               | C5    | 2 call sites greped, both updated  |
```

Deliberately skipped: C4 (no adversary on a two-line change - the C2 red
already demonstrated the behavior change), C8 (no auth surface touched).
Skips named in the gate line, not hidden.

## APPENDIX C - TONE RULES FOR THE VERDICT

- PROVEN: every required row shows a real observation from this session.
- UNTESTED: claim + the exact command the reader can run. Honest and useful.
- Never: "should work", "looks correct", "tests are passing" without paste,
  "verified" without a named command, or a bare "done" with no ledger.
- Confidence language is not evidence. The ledger does not contain adverbs.

## APPENDIX D - VERBAL FORMS (what each phrase commits you to)

| Phrase you emit | What you have committed to showing |
|---|---|
| "done" | the full ledger, 6-8 rows, this session |
| "fixed" | the repro that failed before and passes now |
| "deployed" | a public-path read-back showing the new version |
| "tests pass" | the runner output with exit code |
| "verified" | the command, or the words "untested" |
| "untested" | the exact command the reader can run |
| a number | source file + method to recompute it |

## APPENDIX E - MAINTENANCE

- Re-read this skill when a false-done ships anyway: the gate missed a class
  of check, and the incident gets appended to Appendix A.
- The probe set (C7) grows by one entry per production incident; entries are
  never removed, only marked retired with a reason.
- The mechanism catalog (C3) gains a row per new platform the fleet deploys to
  (new host, new CDN, new store) - a deploy path with no read-back entry is a
  standing gap.
- Quarterly: re-run the gate against its own definition. Introduce a defect on
  purpose and confirm the gate blocks it. A guard that was never seen failing
  on its own defect is an unverified guard (rule 2, Proxy Metric Rules).

