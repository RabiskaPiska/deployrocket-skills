---
name: honestscore
description: Score a deliverable (code / brain doc / website) on the honest, non-optimistic 1-10 rubric - subtractive, evidence-gated, per-dimension, adversarial; code gets performance + concurrency dimensions and a measurable 9.7 bar for small code changes. Use before claiming "done" or inside any "loop until >=X".
alwaysFire: true
---

# honestscore - The Non-Optimistic Quality Gate

## 0. IDENTITY AND CONTRACT

honestscore is the final quality gate. It scores a deliverable the way a hostile,
evidence-demanding reviewer would: subtract from 10, never add, never average, never round up.
Its only loyalty is to the measured truth, not to the effort spent.

The hard rules, in one breath:

1. Every point removed must be justified by evidence (a tool result, a file, a test output).
2. Every point kept must be backed by evidence of at least Tier E2 (Section 5).
3. The deliverable's score is the MINIMUM across dimensions, not the mean.
4. An unmeasured claim is a deduction, never a pass.
5. A self-graded score without a de-correlated adversarial pass is invalid and must be re-run.
6. The loop has a 2-round cap. After the cap, report honestly and stop.

You may not skip a dimension because "it does not apply." If it truly does not apply,
mark it N/A with one sentence of justification; N/A dimensions are excluded from the MIN.
At most 3 of the 8 dimensions may be N/A on any one deliverable. More than 3 means the
deliverable type was classified wrong - go back to Section 2 and reclassify.

## 1. WHEN THIS SKILL FIRES

- Before ANY "done" / "fixed" / "shipped" / "complete" claim is written.
- Inside any "loop until >= X" run, each iteration.
- When godmode, goal, or the Stop hook demands a quality score.
- When a subagent returns a deliverable that will be merged.
- When the user asks "how good is it really", "score this", "is it actually done".

It does NOT fire on: trivial conversational answers, one-line questions, or pure Q&A
with no deliverable. When in doubt, score it - a wasted honestscore is cheaper than an
unverified done-claim.

## 2. STEP 1 - CLASSIFY THE DELIVERABLE

State out loud, in one line, before scoring anything:

```
DELIVERABLE: <type> | STATED REQUIREMENTS: <one line> | DONE MEANS: <one line>
```

Deliverable types and their thresholds:

| Type | Examples | Threshold |
|---|---|---|
| CODE-CRITICAL | auth, payments, crypto, concurrency, data migration, anything security-sensitive | >= 9.7 |
| CODE-SMALL | small diff, single function, isolated fix (roughly < 200 changed lines) | >= 9.5 |
| CODE-GENERAL | features, refactors, scripts, tooling | >= 9.5 |
| DOCS | brain notes, runbooks, reports, README, skill files | >= 9.0 |
| SITE | client sites, landing pages, UI builds | >= 9.0 |
| RESEARCH | research reports, cited syntheses | >= 9.0 |
| MIXED | pick the threshold of the highest-stakes component | highest bar wins |

"DONE MEANS" must name the artifact(s) that prove completion. If done was never defined,
define it now from the user's request - and say that you defined it.

## 3. STEP 2 - THE 8 DIMENSIONS

Score each dimension by subtracting deductions from 10.0. Never start below 10.
Deduction caps are per dimension. A dimension you did not measure earns its full
deduction ladder (Section 8: "could not measure" is NEVER a pass).

Dimension applicability matrix:

| Dimension | Code | Docs | Site | Research |
|---|---|---|---|---|
| D1 Correctness | yes | yes | yes | yes |
| D2 Completeness | yes | yes | yes | yes |
| D3 Robustness | yes | n/a ok | yes | n/a |
| D4 Security | yes | n/a | yes | n/a |
| D5 UX | n/a ok | clarity only | yes | readability |
| D6 Performance | yes | n/a | yes | n/a |
| D7 Concurrency | yes (if shared state) | n/a | n/a | n/a |
| D8 Documentation | yes (docs of the change) | yes (the doc itself) | yes (on-page copy) | yes (citations) |

### D1 - Correctness (deductions 0 to -3)

Does it do exactly what was asked, verified against the stated requirements?

- -0.0: every stated requirement has named, fresh evidence it holds (command output,
  test result, file read back from disk).
- -0.5: one minor requirement is asserted but only self-verified (no tool output).
- -1.0: one requirement unverified, or one edge of the requirement ignored.
- -2.0: a requirement is demonstrably not met (show the failing evidence).
- -3.0: the deliverable does the WRONG thing, or the core claim is contradicted by the
  deliverable's own artifacts (the jury-verdicts failure shape: stored flags contradicted
  by the bundle's own scan output).

Correctness checks that count as evidence, in descending strength:
1. A runnable check whose failing branch you have personally seen fail (test that was
   made to fail once, then pass).
2. A fresh command with exit code captured this session.
3. Reading the produced artifact back (file, HTTP response, DB row).
4. A different machine or parser consuming the output successfully.
A re-read of your own answer counts for NOTHING (the same reasoning that produced an
error re-produces it on re-read - measured repeatedly; external checks catch errors,
read-throughs re-confirm them).

### D2 - Completeness (deductions 0 to -2)

Does it cover the full scope, including the parts that are annoying?

- -0.0: every scope item delivered and evidenced; edge cases and error states covered.
- -0.5: one edge case or error state missing (empty input, zero results, network down).
- -1.0: one scope item missing entirely, or "remaining work" silently dropped.
- -2.0: multiple scope items missing, or scope was quietly narrowed without saying so.

Watch for the classic silent drops: only the happy path tested, only the primary
viewport (desktop but not mobile), only the main locale, only one browser, the
"boring" requirements (logging, rollback, migration) skipped.

### D3 - Robustness (deductions 0 to -2)

What happens when the world misbehaves: bad input, timeout, retry, partial failure?

- -0.0: failure paths named and exercised (at least the top 3 by likelihood).
- -0.5: failure paths handled but never exercised (code exists, no test hit them).
- -1.0: failure paths assumed ("the API always returns 200") without evidence.
- -2.0: any unguarded crash path on real input classes (null, empty, huge, unicode,
  concurrent, replayed).

The fleet's own history is the calibration: deadline-killed sessions looked identical
to quality failures until someone read the session outcome server-side (verdict
adjudication). Robustness means the deliverable distinguishes its own failure modes
instead of collapsing them into one boolean.

### D4 - Security (deductions 0 to -2)

- -0.0: input validated at real boundaries, secrets in env/vault, no injection sinks on
  user input, auth surface checked (expiry, refresh, CSRF, session fixation when touched).
- -0.5: validated but one surface unchecked (e.g. file names, headers, redirect targets).
- -1.0: a real vulnerability class reachable by the deliverable's own usage (SQL string
  concat, unescaped output, eval on input, unbounded upload).
- -2.0: secret in code/logs/history, auth bypass, or the change disabled a security
  control without saying so.

Rule: "Trust internal code and framework guarantees" - do not invent threats for
impossible cases, but DO check every real boundary (user input, external API, file
upload, URL parameter).

### D5 - UX (deductions 0 to -2)

For sites/UI: hierarchy, contrast, spacing, mobile behavior, error states, empty states,
loading states, focus order. For docs: would a new reader get to first success?

- -0.0: all states designed (loading/error/empty), verified at real display size.
- -0.5: states exist but were never looked at rendered.
- -1.0: obvious rough edge a user hits in the first 5 minutes.
- -2.0: primary flow broken or unusable on a real viewport/reader.

The measured-failure rule applies verbatim: judge at the size and context the user
actually sees. A screenshot at 3x zoom or a grep-count of "accessible" attributes is
not UX evidence; open the artifact at real display size.

### D6 - Performance (deductions 0 to -1)

- -0.0: no obvious bottleneck, or measured with a stated method (benchmark, profiler,
  Lighthouse, timing output) and the number is recomputable from files that exist.
- -0.5: unmeasured suspicion of a hotspot in a hot path (N+1 query, per-request
  subprocess, O(n^2) over user data, unbounded memory growth).
- -1.0: known-bad pattern in a hot path left in place.

A number written down must be recomputable from files that exist, with the method
stated. A fabricated number that reaches a report is an automatic D1 deduction as well
(it is a correctness defect, not just a performance one).

### D7 - Concurrency (deductions 0 to -2; code with shared state or parallel work only)

- -0.0: shared state identified, locking/atomicity correct, `-race` (Go) or equivalent
  sanitizer run and pasted, no data race on the exercised paths.
- -0.5: locking present but untested under load; no race run.
- -1.0: a data race or TOCTOU reachable in normal operation.
- -2.0: shared mutable state with no synchronization at all, or a limit-enforced
  endpoint with no concurrency defense.

This dimension is N/A only when the deliverable provably has no shared state, no
goroutines/threads/tasks, and no parallel writes - say why in one line.

### D8 - Documentation (deductions 0 to -2)

The change explains itself: commit message, doc updates, runbook entry for any gotcha,
honest "untested" labels where true.

- -0.0: artifact documented at the level its audience needs; any root-cause hit during
  the work was written to the brain/runbook (symptom -> cause -> fix).
- -0.5: documented but the runbook/brain entry was skipped.
- -1.0: non-obvious behavior with no explanation; the next person must re-derive it.
- -2.0: the deliverable contradicts its own docs, or docs describe a feature that was
  not actually built (the placeholder-skill failure mode).

## 4. STEP 3 - EVIDENCE TIERS

Every dimension score must cite its evidence tier. Tiers:

- E0 - NONE: nothing but intuition. Contribution to the score: zero. A dimension scored
  on E0 evidence is capped at 5.0.
- E1 - SELF-CLAIM: "it works", "I checked", "should be fine". Uncited reasoning. Caps
  the dimension at 7.0.
- E2 - PASTED OUTPUT: a real tool result from THIS session quoted in the working notes:
  command + output + exit code. Full credit possible.
- E3 - FRESH INDEPENDENT CHECK: a check that could have come out the other way and was
  run against the artifact as-is: a test whose failing branch you saw fail, a second
  machine reading the result back, a parser consuming the output, a benchmark rerun.
- E4 - DE-CORRELATED ADVERSARIAL PASS: Section 6. Required for any score >= 9.0.

Pasted-output rules (E2):

1. The paste must include the invocation, the output, and the exit status. An output
   with no invocation is unverifiable (half credit at most).
2. Output must be from this session. Reusing output from a previous session is E1
   (you cannot prove the artifact has not changed since).
3. Exit codes read through a pipe are a known trap: a gate piped into `tail` can return
   0 on a rejected input. Capture the exit code at the source, unpiped.
4. A screenshot counts only for visual claims, must be taken at the size the user sees,
   and must include the scroll position where the claim lives (screenshots taken
   without scrolling produced real false passes here).
5. A passing truth table proves the rules fire, not that input can reach them. For any
   gate/check you built, attack it once: try to get bad input past it. An unattacked
   gate is E2 evidence at best.
6. grep/line counts as evidence: rerun the count yourself in the same session; never
   inherit a count from prose ("the test passes" is E1; "pytest -q -> 42 passed, 1
   xfailed, exit 0" is E2).

Caps summary:

| Tier | Max dimension score |
|---|---|
| E0 | 5.0 |
| E1 | 7.0 |
| E2 | 9.5 (9.7 for a single-dimension spot check with attack evidence) |
| E3 | 10.0 |
| E4 | 10.0 |

## 5. THE MIN-NOT-MEAN RULE (score arithmetic)

```
score = MIN over non-N/A dimensions, truncated (never rounded) to 1 decimal
```

- Two dimensions at 7.0 and 9.5 -> the deliverable scores 7.0. Not 8.2. Not "7-9".
- Never round up: 8.96 is 8.9. Never round to a "clean" number.
- If any binary gate (Section 7) failed: the score is 5.0 or below regardless of
  dimensions, and the verdict is FAIL until the gate is fixed.
- If the adversarial pass (Section 6) was skipped: the score is invalid - report
  "unverified" rather than a number, or run the pass.

Report format for the arithmetic (mandatory lines in the report):

```
D1 10.0 (E3: pytest 44/44, exit 0, failing branch seen pre-fix)
D2 9.0 (E2: scope checklist, mobile check UNMEASURED)
D6 8.5 (E2: hyperfine 12.1ms vs 41ms baseline, n=50)
...
SCORE: min = 8.5 -> FAIL (threshold 9.5)
```

## 6. STEP 4 - THE DE-CORRELATED ADVERSARIAL PASS (mandatory for >= 9.0)

Requirement: at least one check that COULD have come out the other way and that was
not produced by the same reasoning that produced the work. Re-reading your own answer
is worth zero here - the same reasoning that produced the error produces the same
review.

Qualifying checks (any of these; target 6-8 on high-stakes work):

1. A command with an exit code, run against the artifact as-is.
2. A different machine / parser / reader consuming the result back.
3. An adversarial pass by a fresh-context reviewer told to REFUTE the claim, scoped to
   correctness + stated requirements only (an unscoped "find problems" reviewer
   invents defects; scope the brief).
4. A test whose failing branch you have actually seen fail at least once.
5. A recomputation of every number in the deliverable from files that exist.
6. Re-introducing the original defect and watching the gate/guard exit non-zero (a
   guard does not exist until PROVEN to fail on the defect it was written for).
7. Attacking your own gate with 10 minutes of adversarial inputs (escape-hatch hunt).
8. A blind judge that has not seen your reasoning, judging only the artifact.

The adversarial pass is de-correlated when the checker could plausibly have returned a
different verdict than your self-assessment. If you cannot name what each check
checked, you have not done them.

For small code changes, the measured bar is 9.7; the minimum adversarial set is:
(a) the new code path executed once for real, (b) one negative test / attack on it,
(c) one regression check that the old behavior still holds (trace callers), (d) the
diff read once specifically hunting for the exact class of bug the change could
introduce.

## 7. STEP 5 - BINARY GATES (checked before scoring)

Each gate is pass/fail. Any FAIL caps the whole score at 5.0 until fixed.

- G1. Reproducibility: every number/claim in the deliverable is recomputable from files
  that exist, with the method stated. (A number nobody can recompute is treated as
  fabricated.)
- G2. No fabrication: no invented command output, no invented file paths, no invented
  versions/CVEs/prices. Anything not verified is labeled unverified IN the artifact.
- G3. No secrets: no API keys, tokens, passwords, credentials in the deliverable, its
  logs, or its examples.
- G4. Runs/builds: for code, the build/lint/test command was run this session and the
  output was pasted. For docs, the file parses (frontmatter/links/encoding as
  applicable). For sites, the page loads with no console errors.
- G5. Honesty labels: every untested or unverifiable item is explicitly marked
  "untested - needs verification". A hidden unknown is worse than a known gap.
- G6. Scope honesty: the claim does not exceed the requirements ("done" means what the
  user asked for, not an adjacent thing you preferred to build).
- G7. No overclaim language: no "should work", "basically done", "production-ready"
  without the evidence attached. The 60-phrase blocker list from anti-overclaim
  applies.

## 8. MEASUREMENT DISCIPLINE (hard rules from measured failures)

1. A proxy metric may rank variants of the SAME treatment; it may never choose BETWEEN
   treatments. If a scalar metric (variance, edge energy, IoU, coverage %) picked the
   winner, open the artifact and look before accepting the pick.
2. A guard does not exist until PROVEN to fail on the defect it guards against.
   Re-introduce the defect; the guard must exit non-zero. Until then the guard is
   inert.
3. Every number in the deliverable must be recomputable: name the file(s) and the
   command that recompute it. If you cannot, mark the number unverified or delete it.
4. Judge at the size and context the user actually sees (display size, mobile, the
   real terminal width), not the size that was convenient to measure.
5. "Could not measure" is NEVER a pass. If you could not measure, the dimension gets
   its full deduction ladder, and the report says UNMEASURED - counted as a defect,
   not a pass. 0 pass / 1 defect / 2 unmeasured beats a confident guess.
6. When one of these is violated, the fix is a CHECK (a command, a test, a gate), not
   a resolution to be more careful.

## 9. STEP 6 - THE LOOP (2-round cap)

If the score is below threshold:

1. Name the lowest-scoring dimension(s) and their exact defects (evidence-cited).
2. For each defect, emit a ticket (Section 10) and call Skill(ideas) to generate
   candidate fixes; test the best; implement.
3. Re-run honestscore from Step 1 (fresh evidence, previous claims not carried over).
4. Repeat until score >= threshold OR 2 repair rounds have been consumed.

Round definition: one round = one diagnose -> fix -> re-score cycle. Two rounds is the
hard cap inherited from the fleet protocol (jury repair loop: <= 2 rounds). After the
cap:

- Report the final score, the blocking dimensions, and the exact evidence gap.
- State what the user must supply (a credential, a decision, an environment) if the
  blocker is external.
- NEVER fake completion at the cap. An honest "8.6, blocked on X" beats a fake 9.5.

Early stop (allowed): stop below threshold ONLY if 100% honestly certain improvement
is impossible right now; state the blocker + what the user must supply. Otherwise keep
iterating (the standing directive allows 15-20 rounds for a declared 9.5 goal - the
2-round cap here is the per-deliverable repair cap; a goal-level loop may spawn new
approaches via /ideas, which is not the same as re-running the same fix).

Improvement honesty: if round 2 scores LOWER than round 1, stop, revert the last
change if it made things worse, and report. Two failed corrections on the same defect
means the approach, not the patch, is wrong (write the one-sentence diagnosis, then
restart from a sharper spec in clean context - do not keep patching).

## 10. TICKET FORMAT FOR /ideas

When honestscore finds a real defect, it must emit a machine-usable ticket per defect
so the ideas skill can act without re-deriving context:

```
TICKET HS-<n>
SYMPTOM: <what is wrong, observable, one line>
EVIDENCE: <tool output / file / command that proves it - paste the payload>
DIMENSION: D<n> - <name>
SEVERITY: block | major | minor
SUSPECTED CAUSE: <one sentence, may be a hypothesis>
FALSIFIER: <the test that would prove this ticket wrong>
CANDIDATE FIXES: (filled by ideas skill)
STATUS: open | fixed | wontfix
```

Rules: one ticket per defect (no batching "several small issues"), evidence is
mandatory (no ticket without E2+ evidence), and every ticket opened MUST be closed
(fixed, or explicitly accepted as open with reason) before the deliverable can pass.
Open tickets at loop end = the report's REMAINING section.

## 11. REPORT FORMAT (the score block)

Every honestscore run ends with this block, verbatim structure:

```
HONESTSCORE <type> <threshold>
D1 Correctness   X.X  (En: evidence one-liner)
D2 Completeness  X.X  (En: ...)
D3 Robustness    X.X | N/A - reason
...
BINARY GATES: G1 pass | G2 pass | ... (any fail listed with detail)
ADVERSARIAL PASS: <what was run, what it could have found, what it found>
SCORE: <min> -> <PASS|FAIL> vs threshold <T>
DEFECTS: [ticket ids + one-liners]
NEXT: <the single highest-leverage fix, if FAIL>
```

The gate line for the parent session remains the parent's responsibility; honestscore
contributes its score line into the parent's gates line:

```
/honestscore: fired <score>
```

An honestscore invocation that produced no score line did not happen. Reciting the
rubric from memory is NOT a fired honestscore - it is a skip and must be reported as
"skipped: why" in the gates line.

## 12. WORKED EXAMPLE 1 - small code change (bar 9.7)

Requirement: "add retry with backoff to the API client."

Scoring:
- D1 Correctness 9.8 (E3): test written first, seen failing (TimeoutError), then
  passing after the change; callers traced, none affected.
- D2 Completeness 9.5 (E2): edge cases covered; jitter variant untested (-0.5).
- D3 Robustness 9.5 (E2): max attempts respected; the "non-retryable 4xx" path exists
  but no test hits it (-0.5).
- D4 Security 10.0 (E2): no new input surface; grep for secret leakage in retry logs
  clean (pasted).
- D5 UX: N/A (library code; error surface unchanged) - justified.
- D6 Performance 9.8 (E2): backoff caps at 30s, bounded, benchmark unchanged.
- D7 Concurrency 10.0 (E2): client is per-request state; go test -race clean, output
  pasted.
- D8 Documentation 9.0 (E2): changelog entry written; runbook entry for the
  "retry-storm" gotcha NOT written (-1.0), caught by adversarial pass.
- Gates: G1-G7 pass.

MIN = 9.0 -> FAIL vs 9.7. Fix the D8 defect (write the runbook entry, one file),
re-score: D8 10.0, MIN 9.5 -> still below 9.7 -> round 2: address the D3/D2 gaps
(add the 4xx test + jitter test), re-score 9.7 -> PASS. Two rounds, within cap.

## 13. WORKED EXAMPLE 2 - brain doc (bar 9.0)

Requirement: "rewrite the deploy runbook with the current process."

Scoring:
- D1 Correctness 9.5 (E3): every command in the runbook executed fresh; two commands
  corrected (they failed as written; outputs pasted).
- D2 Completeness 9.0 (E2): all 14 steps covered; rollback section present.
- D3: N/A - no code.
- D4 9.5 (E2): no secrets in the file (gitleaks run, output pasted).
- D5 UX: readability - 9.5 (E2: structure checked by reading at real length; tables
  render).
- D6: N/A - no runtime.
- D7: N/A - no shared state.
- D8 Documentation 10.0 (E2): the doc IS the documentation; sources cited.
- Gates: G1 pass (commands recompute), G2 pass (no unverified numbers), G5 pass.

MIN = 9.0 -> PASS at 9.0. Note the honesty requirement: the two corrected commands
MUST be stated in the report, not hidden - corrected-command counts as found defects,
which is the point of the gate.

## 14. ANTI-GAMING RULES

1. You may not review your own high-stakes deliverable with re-reads. External checks
   only (Section 6).
2. You may not raise a dimension score without new evidence; evidence from before the
   last change is stale.
3. You may not drop a dimension to make MIN rise (that is N/A abuse - max 3 N/A).
4. You may not weaken a test to pass. If a test is wrong, say why and fix it openly.
5. You may not claim "could not measure" as a pass. It is a defect, counted.
6. You may not count "it compiles" as correctness evidence.
7. Self-reported flags are distrusted by default: an independent recompute that
   contradicts a stored flag wins (the jury-round-1 rule: independent recompute beats
   stored pass flags, every time).
8. A stub row (no real run behind it) is UNRESOLVED, never counted as pass or fail.

## 15. QUICK REFERENCE CARD

```
1. CLASSIFY  type + requirements + done-means
2. SCORE     8 dimensions, subtract from 10, cite evidence tier per dimension
3. GATES     G1-G7 binary; any fail caps at 5.0
4. ADVERSARY de-correlated external pass (mandatory for >= 9.0)
5. SCORE     MIN, truncate, never round up, never average
6. LOOP      fix lowest dimension -> re-score; 2-round cap; then honest report
7. REPORT    score block verbatim + ticket list; "fired N.N" into gates line
```

Thresholds: docs/site 9.0 | code 9.5 | security-critical code 9.7.
N/A cap: 3 dimensions. Repair cap: 2 rounds. Tiers: E0 cap 5.0, E1 cap 7.0, E2 9.5.

## 16. INTEGRATION

- godmode: honestscore is a mandatory member of the chain; godmode's gate line reports
  the score.
- ideas: every defect found here becomes a ticket (Section 10) and ideas generates +
  scores fixes.
- goal: the loop engine calls honestscore each iteration; its stop condition consumes
  the score.
- session-monitor-auto: the watchdog's completion check (Section 7 of that skill)
  requires the honestscore MIN gate before declaring GOAL ACHIEVED.
- anti-overclaim: this skill's report must not contain blocked phrases; anti-overclaim
  runs as gate G7.
- subagents: for big deliverables, the adversarial pass is spawned as a fresh-context
  subagent (never run in the authoring context).

---

# APPENDIX A - SITE RUBRIC (websites / landing pages)

When the deliverable is a site, D1-D8 resolve as follows. Everything here requires the
artifact open in a real browser at real viewports; a screenshot of the top of the page
is not evidence.

### A.1 D1 Correctness for sites

- Every link resolves (crawler or curl the hrefs; count 404s).
- Forms submit and produce the documented side-effect (lead row, email, 200 response).
- Fonts/images load (no console 404s); favicon present.
- Meta tags match the page (title, description, og tags, canonical).
- Lighthouse run pasted (performance + SEO + best practices + a11y categories).
- Text content matches the client brief (name, hours, address, phone correct).

Deductions: -0.5 per broken link or 404 asset up to -1.5; -1.0 broken form or broken
anchor; -1.0 wrong client facts; -0.5 missing/contradictory meta.

### A.2 D3 Robustness for sites

- Mobile 375px and 768px checked in a real browser (device if available; at minimum
  devtools emulation with real scrolling, not just a resize).
- Landscape orientation does not break the layout.
- Slow-3G and cached-repeat loads both render acceptably.
- Images have explicit dimensions or a reserved aspect ratio (no CLS).
- Long content (2x the copy) does not overflow containers.

### A.3 D4 Security for sites

- Forms validate server-side; no raw HTML output of user input.
- No secrets in client bundles (grep the built assets for key-shaped strings).
- Headers present where the host controls them: CSP, X-Content-Type-Options,
  Referrer-Policy at minimum; document any control you did not set.
- Third-party scripts pinned and explained (each is an XSS surface).

### A.4 D5 UX for sites (the anti-slop pass)

Judge at real display size. Banned as evidence: "looks fine", the template's
defaults, a metric score alone. Required: you looked at the page, scrolled it, at
mobile and desktop widths.

Checklist (each unchecked box is -0.5 on D5):

- [ ] One clear primary action per screen; hierarchy reads top-to-bottom.
- [ ] Contrast passes AA for body text (measure, do not eyeball).
- [ ] Spacing is consistent (one spacing scale, not five).
- [ ] Mobile nav works one-handed; tap targets >= 44px.
- [ ] Every image has alt text; icons have labels.
- [ ] Error and empty states exist and are written in product voice, not dev voice.
- [ ] No layout shift on load (CLS < 0.1 measured, not assumed).
- [ ] The page does not look like every other AI-generated template (font, color,
      layout carry at least one deliberate, non-default choice).

### A.5 D8 for sites

- Deploy pipeline documented; rollback path named.
- What was verified vs untested is stated on the delivery report (never implied).

---

# APPENDIX B - CONCURRENCY DEEP CHECKLIST (D7)

Run these in order; each unchecked item is a deduction with its number.

- B1. Enumerate shared state: globals, singletons, files, caches, DB rows, queues.
- B2. For each, name the access discipline: lock, atomic, single-writer, or
      immutable-by-design. "It is probably fine" is not a design.
- B3. Race detector run on the actual concurrent path (go test -race with a test that
      exercises concurrency; -0.5 if the suite has zero concurrent tests).
- B4. Check the four classics on any limit-enforced or write path: double-submit,
      retry-after-partial-failure, read-modify-write without transaction, and
      check-then-act (TOCTOU).
- B5. Timeouts: every blocking external call has one; nothing waits forever.
- B6. Shutdown: workers drain or are cancelled, nothing leaks on error paths.
- B7. Idempotency: replaying the same request twice is safe, or replay is impossible
      by construction (and you can say how).

Deduction map: any B2 gap -1.0; B3 race-detector missing on concurrent code -0.5;
a confirmed race or TOCTOU reachable in production usage -1.0; unbounded wait or leak
-0.5 each.

---

# APPENDIX C - EVIDENCE PITFALLS (measured failure catalog)

Each of these produced a false "done" in real fleet sessions. All are treated as E1
(no matter how confident the wording) unless fixed as described:

- C1. Exit code read through a pipe: `cmd | tail` returns the last command's status.
      Fix: capture `${PIPESTATUS[0]}` / run unpiped, or check the artifact itself.
- C2. `pgrep` matching its own command line / the calling ssh's cmdline (the pkill
      self-kill class). Fix: bracket-trick patterns (`pkill -f 'proof[.]py --host'`),
      and verify by side-effect, not by process-list match alone.
- C3. A count taken from off-canvas or unscrolled content (CSS overflow hidden
      elements still match selectors). Fix: verify in rendered output at real viewport.
- C4. Judging from a downscaled thumbnail or zoomed render. Fix: render at the exact
      display size and look.
- C5. Stored flags trusted over fresh computation (jury round 1: stored
      gate_line_present=true contradicted 17/20 by the independent scan). Fix: the
      independent recompute always wins; stored flags are claims, not evidence.
- C6. Excerpt truncation hiding the checked artifact (the 4000-char head vs the
      1500-char tail problem). Fix: parse the full artifact; if you must excerpt,
      anchor the excerpt at the evidence.
- C7. `$?` / `RC=` interpolation inside a double-quoted remote or PowerShell string
      expanding locally. Fix: single-quote the remote script or read the exit-bearing
      artifact itself.
- C8. Duplicate-inflation: byte-identical fire/evidence lists across unrelated items
      (copy-paste rows). Fix: dedupe by id and check cross-row similarity before
      trusting any aggregate.
- C9. Historical state read as current state (a hook scanning the last N ledger lines
      flags old P1s after restart). Fix: scope checks to entries created after run
      start.
- C10. Epoch-scale or zero durations accepted without comment (56.9-year durations).
      Any impossible number in a results file invalidates that row: treat it as
      UNMEASURED.

---

# APPENDIX D - SCORING WALKTHROUGH FOR MIXED DELIVERABLES

A mixed deliverable (site with a form backend) takes the highest-stakes threshold
(a security-critical form handler pulls the bar to 9.7). Score BOTH halves; the MIN
applies across all dimensions of both halves. Do not average the halves, do not score
them separately and report two numbers - there is one score.

When one half is trivially small (a 5-line config change inside a big doc rewrite),
still score it; trivial does not mean exempt, it means the dimensions that are N/A
are N/A with a reason.

---

# APPENDIX E - WHAT "FIXED" MEANS BEFORE RE-SCORE

A defect is fixed only when:

1. The fix is in the artifact (not planned, not "will fix next round").
2. The original falsifier from the ticket now passes (run it, paste it).
3. The fix did not break a neighboring check (re-run the checks adjacent to the change
   - at minimum, the ones covering the same file or feature).
4. The fix did not weaken a test or gate to manufacture the pass (diff the test if
   touched; any touched test needs a justification line in the report).

Re-scoring after a fix uses fresh evidence for the whole dimension, not just the fixed
defect: the dimension score may go up by at most the severity of the fixed defect
(block +1.5, major +1.0, minor +0.5), never by assumption.

---

# APPENDIX F - THE HONEST UNCERTAINTY ESCAPE HATCH

If after real effort a dimension cannot be measured (no environment, no credentials,
no browser), the honest output is:

```
D6 Performance: UNMEASURED - no benchmark available in this environment.
   Command the user can run: <exact command>
   Provisional score NOT issued. Verdict: UNVERIFIED.
```

An "UNVERIFIED" verdict is always acceptable. A fabricated number is never.
The one-sentence honest uncertainty statement beats a confident guess - this rule is
the skill's reason for existing.

---

# APPENDIX G - DOC AND SKILL-FILE RUBRIC (brain docs, SKILL.md, runbooks)

Docs score against D1/D2/D4/D5/D8 (the others are N/A with a reason). Extra
doc-specific checks, each of which has produced real fleet defects:

- G1. Frontmatter parses: `---` block with name/description on the FIRST lines; a
  missing frontmatter has provably dropped skills from the runtime index (the
  skill-dag-planner defect: absent from the available-skills index until frontmatter
  was added).
- G2. ASCII-clean: no em-dashes, smart quotes, box-drawing, or Unicode Tags
  characters (U+E0000 block has crashed runtimes). Check with a byte-level scan, not
  a console read: PowerShell consoles render mojibake for clean files (display
  artifact) and can hide dirty bytes (the inverse error). Both directions are traps.
- G3. No padding: raw line count minus `<!-- padding line N -->` comments equals the
  real content count. A file that passes a 500-line gate via padding comments is a
  FAIL on D2 (fake completeness) and G7 (overclaim).
- G4. Duplicate-block scan: repeated identical reference blocks (the 49x duplication
  defect) count as padding, not content.
- G5. Every referenced file, skill id, and URL resolves; self-references in
  integration lists are a generation defect (flag, -0.5 on D2).
- G6. Advertised protocols exist: if the catalog description promises an 8-phase
  protocol, the 8 phases must be in the file (the deep-test-all failure: description
  rich, body hollow).
- G7. One file, one protocol: two concatenated versions with no version banner is a
  -1.0 D2 defect (fresh readers get two conflicting mechanisms for the same job).
- G8. Dead schema/example URLs are -0.5 each (verify by fetching, once).

A doc passes D8 only when a fresh agent could adopt it from the file alone - no
tribal knowledge, no "ask the owner" steps for things the file should carry.

---

# APPENDIX H - SCORING ARITHMETIC EDGE CASES

- H1. Two dimensions tie at the minimum: report the tie, the score is that value.
- H2. N/A dimensions are excluded from MIN entirely (they do not drag the min down,
  and they do not raise it).
- H3. All dimensions 10.0 with no adversarial pass run: the score is INVALID, not
  10.0. Print "UNVERIFIED" instead. Perfection requires the E4 pass; only measured
  imperfection can claim >= 9.5 without it.
- H4. Score exactly at threshold: PASS (>= is inclusive).
- H5. A binary gate failed: verdict FAIL regardless of dimension scores; the score
  block prints the gate failure as the reason and the dimensions are informational
  until the gate is fixed.
- H6. Stub rows / unmeasured sub-artifacts inside a deliverable (copied evidence,
  dur=0.0 rows): they are UNRESOLVED. They may not count as pass OR fail; the
  deliverable's verdict is capped at UNVERIFIED until they are re-run.
- H7. Rounding: truncate. 8.99 prints 8.9. If the arithmetic lands on a threshold
  only via rounding up, it is below the threshold.

---

# APPENDIX I - MINIMAL SCORE BLOCK TEMPLATE (copy per run)

```
HONESTSCORE <type> | threshold <T> | round <N>
D1 X.X (En: ...) | D2 X.X (En: ...) | D3 X.X/N-A | D4 X.X (En: ...)
D5 X.X/N-A | D6 X.X (En: ...) | D7 X.X/N-A | D8 X.X (En: ...)
GATES: G1 ok | G2 ok | G3 ok | G4 ok | G5 ok | G6 ok | G7 ok
ADVERSARIAL: <check run> -> <found / clean>
MIN: X.X -> <PASS|FAIL|UNVERIFIED>
DEFECTS: HS-1 <one-liner> [open]; HS-2 <one-liner> [fixed, falsifier rerun ok]
```

The block is the evidence trail. A session gate line that says "honestscore: fired"
without a score block is a self-report, treated as E1 by any later audit.

---

# APPENDIX H - FREQUENTLY RATIONALIZED CLAIMS (and their correct treatment)

| Claim made by the author | Correct treatment |
|---|---|
| "It compiles / lints clean" | Necessary, not sufficient. Zero D1 evidence by itself. |
| "The tests pass" without output | E1. Ask for the paste or run it yourself. |
| "I reviewed it carefully" | E1. Re-reads are not adversarial passes. |
| "It should work in production" | Overclaim phrase (G7). Score D1 as if untested. |
| "The logic is obviously correct" | Intuition = E0. Cap the dimension at 5.0. |
| "All edge cases are handled" | Name three and show one exercised, or -1.0 D2. |
| "Performance is fine for our scale" | E1 unless a method + number exists (recomputable). |
| "I'll add tests later" | D2 -1.0 minimum; tests-after is untested code. |
| "It worked on my machine" | E2 at best, single-environment. State the environment. |
| "The user can test it" | Acceptable ONLY as the UNVERIFIED escape hatch output. |
| "Minor issues only" | Severity does not change the MIN rule. Score still MIN. |
| "Previous round scored 9.5, nothing changed since" | Evidence rots with every change. Re-evidence or keep the old score and state nothing changed. |

---

# APPENDIX I - DECISION FLOW (compact)

```
START
  -> classify deliverable, set threshold
  -> run binary gates G1-G7
       any fail -> fix gate -> re-run gates (does not consume a repair round
       unless the fix touches the deliverable)
  -> score 8 dimensions with evidence tiers
  -> adversarial pass (mandatory for >= 9.0 verdicts)
  -> score = MIN, truncated
  -> if PASS: emit score block, done
  -> if FAIL:
       ticket every defect -> Skill(ideas) -> test best fix -> implement
       -> round++ ; round > 2 ? honest report + blocker : re-score
  -> if UNMEASURED anywhere and no path to measurement:
       verdict = UNVERIFIED + exact command for the user
```

---

# APPENDIX J - WORKED EXAMPLE 3 - research / brain doc with adversarial subagent

Requirement: "2026 state of the fleet sync chain, with a go/no-go recommendation."

Round 1 scoring:
- D1 Correctness 7.0 (E2): 12 of 14 claims cite a fetched page; 2 claims rest on
  memory (versions). -1.0.
- D2 Completeness 9.0 (E2): the promised CVE table covers 2024-2026; one promised
  appendix absent. -1.0.
- D4 Security: N/A (no code).
- D5 Clarity 9.5 (E2): structure verified by a read at full length.
- D8 Documentation 9.0 (E2): citations present; two sources conflict and the conflict
  is not flagged. -0.5.
- GATES: G2 flagged (two uncited claims).
- ADVERSARIAL: fresh-context subagent, mandate REFUTE, returns: one claim's source
  page says the opposite of the report (fabrication risk caught).
- MIN 7.0 -> FAIL.

Round 1 fix: the two claims web-verified and rewritten; the conflicting sources
flagged with both readings. Round 2: all claims cited, adversarial pass re-run, no
refutation survives. MIN 9.0 -> PASS (docs bar). The repair is reported, including
the originally wrong claim - an honest score report names what was wrong, not only
what is now right.

---

# APPENDIX K - INTERACTION WITH THE GATES LINE

When the parent session must emit a gate line of the form:

```
gates -- ... | /honestscore: <fired N.N|skipped: why> | ...
```

honestscore is "fired N.N" ONLY when a real Skill invocation happened this turn AND
this skill's score block was produced (even if the verdict is FAIL or UNVERIFIED -
fired means executed, not passed). Reporting "skipped: why" is always honest; the
only crime is silent omission.
