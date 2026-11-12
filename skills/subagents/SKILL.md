---
name: subagents
description: Decide HOW MANY subagents to spawn for ANY incoming task - every task runs a 5-30 agent crew (hard floor 5, hard ceiling 30, owner-set) scaled by importance; score importance/complexity, decompose into disjoint mandates, dedup, wave-schedule, orchestrator-owned merge, adversarial verification. Extra seats fill by ROLE, never by duplicated scope.
alwaysFire: true
---

# subagents - The Crew Protocol

## 0. IDENTITY AND HARD RULES

Every substantive task runs a crew of subagents. The floor and ceiling are not
suggestions and not negotiable in-session:

- HARD FLOOR: 5 subagents per substantive task.
- HARD CEILING: 30 subagents per task.
- Seats are filled by ROLE, never by duplicated scope.
- The orchestrator (you) is seat 0 and never counts toward the floor.
- Trivial/conversational tasks skip the crew entirely (answer directly); the floor
  applies to substantive work only.

Corollary rules the protocol enforces:

1. No two seats share a mandate. If two seats would read the same files and answer
   the same question, merge them and give one seat the freed work.
2. A seat you could do faster inline than you can write the mandate for, is a seat
   that should not exist - but the floor still stands: find the five things that
   genuinely benefit (verification, adversarial pass, recon, cross-check, build).
3. You may exceed the floor for good reasons; you may never exceed the ceiling.
4. If spawning is impossible (no subagent tool), you do the work yourself and say so
   - the floor is about thoroughness, not ritual.

## 1. WHEN THIS SKILL FIRES

- Any substantive build / fix / research / audit task (not a one-line change, not a
  conversational answer).
- Whenever godmode or goal activates (they mandate the crew).
- The user says "spawn a crew", "use subagents", "fan out", "parallelize".
- A task decomposes into 2+ independent work units with file or domain separation.

Skip conditions: trivial tasks, tasks where the only parallelizable work is a single
read, or tasks where the user explicitly ordered a solo run.

## 2. IMPORTANCE SCORING -> SEAT COUNT

Score the task 1-10 on importance/complexity BEFORE deciding seats:

| Factor | Weight | How to judge |
|---|---|---|
| Blast radius | 0-3 | what breaks if the output is wrong (prod > client > scratch) |
| Breadth | 0-3 | how many distinct knowledge domains / file areas it touches |
| Verification need | 0-3 | how expensive a wrong "done" is to discover later |
| Time pressure | 0-1 | parallelism actually buys wall-clock time here |

Score -> seats:

| Score | Crew size |
|---|---|
| 0-2 | 5 (the floor - always at least the five core seats) |
| 3-5 | 6-10 |
| 6-8 | 8-15 |
| 9-10 | 15-30 (ceiling) |

The five core seats are mandatory at every crew size (Section 4): recon,
cross-check, build, verify, adversarial. Extra seats extend these roles by DOMAIN
(one seat per domain/area), never by duplicating the same role twice on the same
scope. If the task has fewer than five natural units of work, the extra seats take
the verification/adversarial burden: split the verification by concern (correctness
verifier, security verifier, regression verifier) rather than inventing fake work.

## 3. TASKLIST-FIRST (write it before spawning anything)

Before the first seat launches, write the tasklist to disk. No seat spawns until the
file exists. Default path:

```
./tasklist.md (or the run's out dir: <run-slug>/tasklist.md)
```

Required columns:

```
| ID  | Seat        | Role        | Mandate (one sentence)                | IN  | OUT | Wave | Status | Evidence |
|-----|-------------|-------------|---------------------------------------|-----|-----|------|----------|----------|
| S1  | recon       | map inputs  | enumerate files/sources + return map  | ... | ... | 1    | pending  | -        |
```

Column definitions:

- ID: stable identifier (S1, S2, ...). Session IDs get attached when spawned.
- Seat: role name (recon / cross-check / build / verify / adversarial / domain role).
- Mandate: ONE sentence, unambiguous, with the success condition inside it. If you
  cannot write the mandate in one sentence, the seat is overloaded - split it.
- IN: exact files/paths/URLs the seat may read. Nothing else.
- OUT: the exact file path or message the seat must produce. Every seat produces a
  written OUT - "told me in chat" is not an artifact.
- Wave: the scheduling wave (Section 6).
- Status: pending | running | delivered | failed | killed | absorbed.
- Evidence: the real side-effect citation (file + bytes, count, commit hash, HTTP
  status). Never "done".

The tasklist is the single source of truth. The orchestrator updates it; seats never
write to it (they report through their OUT). Every beat of the monitor (see
session-monitor-auto) reads this file.

## 4. SEAT ROLES

### 4.1 RECON

Purpose: wide reads so nobody else burns context on raw exploration.
- IN: the task statement, pointers to the codebase/docs/vault areas.
- OUT: a distilled map (target 1-2k tokens, hard cap ~2k): file inventory with
  one-line summaries, key interfaces, existing patterns to mirror, landmines found,
  and a recommended file list for the build seats.
- NEVER dumps raw file contents into its report. Distill or die.
- Multiple recon seats split by area (frontend/backend/docs/external sources).

### 4.2 CROSS-CHECK

Purpose: verify facts, versions, claims against live/external sources.
- IN: the specific claims to verify (a list, not "check everything").
- OUT: per-claim verdict: CONFIRMED (source cited) / CONTRADICTED (what the source
  actually says) / UNVERIFIABLE (what was tried).
- Cross-checks are mandatory for: versions, prices, rate limits, CVEs, "latest/best
  in 2026" claims, and any number that will appear in a report.
- Rule from the fleet: the independent recompute wins over stored claims - a
  cross-check that finds a contradiction beats any stored flag.

### 4.3 BUILD

Purpose: produce the artifact changes (code/docs/site work).
- One build seat per disjoint file-set or worktree. If two build seats must touch
  the same file, they are one seat (or you need the worktree single-writer protocol).
- OUT: the changed files plus a build log (compiler/test/lint output pasted).
- A build seat never marks its own work verified. It reports what it ran; the verify
  seats re-run it.

### 4.4 VERIFY

Purpose: re-run, re-check, and try to break the merged result.
- VERIFY seats run the checks the build seat claims passed: build, tests, lint, the
  runnable pass/fail check named in the spec. Fresh execution, not trust.
- Split verification by concern when crew size allows: correctness verifier,
  regression verifier (callers/dependents), security verifier.
- OUT: check-by-check output with exit codes. "It works" without a command is a
  failed verification.

### 4.5 ADVERSARIAL

Purpose: the de-correlated refutation pass.
- Spawned fresh (no authoring context), mandate: REFUTE the deliverable against the
  stated requirements ONLY. An unscoped "find problems" reviewer invents defects and
  drives over-engineering - the mandate names the requirements and says "refute
  these, do not invent taste complaints."
- OUT: refutation list with evidence, or "no refutation survives" with what it
  tried.
- Minimum one adversarial seat on every substantive run. The honestscore
  de-correlated pass (Section 6 of that skill) is satisfied by this seat.

### 4.6 EXTENSION SEATS (how extra seats fill)

Extra seats extend by ROLE. Valid extensions:
- Domain specialists when the task spans domains (one per domain: SEO, security,
  design, infra...). Each still has a disjoint mandate.
- Additional verifiers split by concern (Section 4.4).
- Additional cross-checks split by claim-set.
- A dedicated merge-prep seat for large merges (collect + dedupe evidence bundles).
- A monitor-relay seat when the orchestrator cannot beat-watch itself (rare; the
  session-monitor-auto protocol normally covers this).

Invalid extensions (banned): a second seat with the same mandate "for speed"
(duplicate scope), a seat whose OUT duplicates another seat's OUT, a seat with no
verifiable OUT.

## 5. PROMPT CONTRACT (what every seat gets)

Every seat prompt contains, in order:

1. CONTEXT: the minimal facts the seat needs (files, paths, decisions already made,
   constraints). Never the whole task history. Never junk - context pollution causes
   tool-hallucination and dropped instructions.
2. MANDATE: one sentence with the success condition.
3. OUT: the exact file path to write and its required shape.
4. EVIDENCE RULE: "Every claim in your OUT needs evidence from this session: command
   output, file content, HTTP response. No evidence = say UNVERIFIED."
5. NO-ASK RULE: "Do not ask questions back. Make the call with the stated
   constraints, mark assumptions in the OUT, deliver."
6. SCOPE FENCE: "Do X only. Do not touch <paths>. Do not refactor beyond the
   mandate."
7. SEARCH BUDGET NOTE: point the seat at the right search tools; the shared built-in
   search budget drains fast under fan-out.

Wave-1 seats additionally get: the goal, the DoD criteria, and the abort criteria.

## 6. WAVE SCHEDULING

Decompose into a DAG. Assign each seat a wave:

- Wave 1: recon + cross-check (independent lookups, no dependencies).
- Wave 2: build seats (need wave-1 outputs).
- Wave 3: verify seats (need the build).
- Wave 4: adversarial (needs a stable artifact to attack).
- Merge and re-verify: after each wave, before the next launches.

Rules:

- 5-8 subagent launches per message. One wave = one message with N subagent calls.
  Never dribble seats out one per turn (that is how a 5-minute task eats an hour).
- Max concurrency is bounded by the harness (and by provider rate limits): 5-8 in
  flight is the working number; if seats return errors that look like rate limiting,
  drop to 3-4 in flight and continue (a dropped seat is rescheduled, not forgotten).
- Every launch in a wave is in a SINGLE message: parallel fan-out, not sequential.
  Sequential launches of independent seats are a defect in this protocol.
- Waves unlock only on verified delivery: a wave-2 seat launches when its wave-1
  dependencies are DELIVERED WITH EVIDENCE in the tasklist, not when they merely say
  done.

## 6.1 SINGLE-MESSAGE FAN-OUT (the mechanical rule)

One message MAY contain multiple subagent calls. The protocol:

- Write the tasklist row for every seat BEFORE the message that launches them.
- Launch each seat's subagent call in the same message, background=true when the
  seats run long or when you need to keep orchestrating, foreground when you truly
  cannot proceed without them.
- Each seat gets a DISTINCT OUT path. Two seats writing one file = clobbered
  results (the fleet hit this with concurrent proof shards; the fix was per-shard
  output paths - apply it universally).
- After launching, arm the monitor loop (session-monitor-auto) - do not sit idle.

## 6.2 DEPTH LIMIT

- Depth 1 only: seats do not spawn their own seats. If a seat believes it needs a
  sub-crew, it returns that need in its OUT and the orchestrator schedules it.
- Rationale: nested fan-out causes context explosion, loss of mandate fidelity, and
  un-auditable evidence chains. The orchestrator stays the single scheduler.
- Exception: none. If a seat spawns a subagent, treat its output as suspect and
  re-verify.

## 7. KILL CRITERIA (when to stop a seat)

Kill / reassign a seat when ANY of:

- 3 same-class probes: the seat has made 3 attempts of the same class (same tool,
  same approach, same error class) without new information. That is a loop; kill it.
- The seat's OUT is empty or UNVERIFIED-only after 2 beats.
- The seat's output diverges from its mandate (wrong direction, semantic drift).
- The seat asks questions back after being told the no-ask rule, twice.

Kill procedure (never silent): mark the tasklist row killed with the reason, log
the evidence, and either reassign (Section 8) or absorb the scope. The kill is
recorded in the runlog with the seat's last verified side-effect.

Timeouts are not quality failures: a seat killed by wall-clock (deadline) after a
verified-productive run is treated as TIMEOUT, distinct from FAIL - check the
session's terminal outcome before deciding (the fleet's verdict-adjudication rule:
fires + evidence outlive the harness's clock).

## 8. MERGE RULES (orchestrator-owned)

- ONE writer merges: the orchestrator. Seats never merge each other's work.
- EVIDENCE WINS CONFLICTS: when two seats' outputs contradict, the one with the
  stronger evidence wins (E4 > E3 > E2 > E1 > E0 by the honestscore tiers). Stored
  claims lose to fresh recomputation.
- If evidence is EQUAL on both sides: prefer the one whose method was stated and
  reproducible; if still tied, run the tie-break check yourself (one command that
  distinguishes them) before merging either.
- Merge order: verified rows first; UNVERIFIED rows merge only with an explicit
  "UNVERIFIED" label carried into the merged artifact; failed rows never merge.
- Provenance stays attached: merged content keeps a pointer to which seat/OUT it came
  from, so a later audit can re-check it without archaeology.

## 8.1 VERIFY-AFTER-MERGE

Merging is not completion. After merge:

1. Re-run the verification checks against the MERGED artifact (the merge itself can
   introduce conflicts neither seat saw).
2. Re-run the named end-to-end check from the spec.
3. If any check fails: fix or re-dispatch a build seat on the failing slice only -
   do not re-run the whole crew for a one-file fix.
4. Then (and only then) run honestscore on the deliverable. The crew's work is not
   done until the honestscore MIN gate passes.

## 9. FAILURE POLICY

Per failed seat, in order:

1. RETRY <= 2: re-dispatch the same mandate with the failure evidence attached
   ("attempt 1 failed with X; avoid that path by doing Y"). Two retries max.
2. REASSIGN: give the mandate to a different seat (fresh context, corrected prompt -
   the failure may be context pollution, and a clean seat is the cheapest cure).
3. TAKE IT YOURSELF: if a third attempt would cost more than doing it inline, the
   orchestrator does the work inline and says so in the report.
4. NEVER: silently drop a mandate, or merge an unverified OUT to hit a deadline.
   A dropped mandate discovered later is the orchestrator's defect, not the seat's.

Failure classification (attach to the tasklist row):
- INFRA: provider error, timeout, empty response -> retry, usually succeeds.
- BAD: seat delivered something off-mandate -> reassign with a sharper fence.
- RETRY-CLASS: transient error class -> same seat, retry once.
- HIT: delivered with evidence -> mark DELIVERED, stop rechecking.

## 9. CONTEXT DISTILLATION (what comes back)

Every seat's OUT follows the distillation contract:

- 1-2k tokens maximum summary (hard cap ~2k).
- Format: FINDINGS (bulleted, each with evidence pointer) / FILES TOUCHED /
  ASSUMPTIONS / OPEN QUESTIONS.
- No raw file dumps. The orchestrator reads the OUT, never the raw sources, unless
  the finding is load-bearing and short.
- If a seat returns a wall of raw text, the orchestrator distills it into the
  tasklist before proceeding - raw reads do not enter the main context twice.

## 10. DO / DON'T

| DO | DON'T |
|---|---|
| Write the tasklist before launching | Launch seats from memory |
| 5-8 seats per message, one wave each | Dribble seats out over many turns |
| Disjoint mandates, filled by role | Clone the same mandate into 5 seats |
| Verify after merge, fresh execution | Trust the build seat's own "pass" |
| Retry <= 2, then reassign, then absorb | Let a failed seat stall the wave |
| Keep seat outputs on disk with provenance | Merge from chat memory |
| State method + evidence in every merge | Merge two contradictory claims as-is |

## 11. ANTI-PATTERNS

1. The 30-seat theater: spawning 30 seats for a 2-file change. The ceiling is a
   limit, not a target. Fill by role, not by number.
2. The hollow crew: 5 seats all doing recon because decomposing felt hard. Build
   seats exist; the artifact needs builders and verifiers, not five maps.
3. Trust-the-seat merges: merging because the seat said "done". The verify wave
   exists because self-reported done is the single most common false positive.
4. The serial fan-out: launching 6 seats in 6 messages. Single message, one wave.
5. Depth creep: seats spawning seats. Orchestrator schedules; seats report.
6. Junk context: pasting the whole repo into every prompt. Seats get maps and
   paths, not raw dumps.
7. Dropping the adversarial seat "because we verified already". Verification and
   refutation are different checks; the second one is what catches the same-reasoning
   blindspot.

## 12. WORKED EXAMPLE (compressed, from a real run)

Task: verify and close out a 20-task proof run across two hosts, evidence-gated.

- Importance: prod evidence, 3 domains (harness, fleet config, verdicts) -> score 7
  -> crew of 8.
- Tasklist first: 8 rows with IN/OUT paths written before any launch.
- Wave 1 (5 launches, one message): S1 recon harness state on host A; S2 recon host
  B; S3 cross-check gate-format spec vs live files; S4 cross-check provider/auth
  constraints; S5 recon the evidence dir (what exists vs what the brief claims).
- Wave 2 (3 launches): S6 merge/rebuild results from fresh REST harvest (host A);
  S7 same for host B; S8 adjudicate pass=false rows against the ADJ rule
  (fires>=7 + gate line + terminal outcome).
- Wave 3 (2 launches): S9 independent strict re-scan (min_fields=14); S10 verify
  provenance (each credited session post-dates the restart; dedupe fires by id).
- Merge: orchestrator merges S6+S7 into one results table; conflicts (S10 found a
  truncated-excerpt false negative) resolved BY EVIDENCE - full-text re-parse wins
  over the excerpt.
- Verify-after-merge: recompute the headline number from the merged file (independent
  recompute = 20/20, matching, flag_mismatches=0).
- Adversarial: 1 seat mandated to refute the 100% claim; it attacked the provenance
  chain and survived.
- Result written to the runlog with per-seat evidence; honestscore run on the report.

Total: 2 messages of launches, 8 seats, 0 duplicated scopes, every row closed with
evidence or explicitly marked UNRESOLVED (the stub rows were re-run, never counted
on trust).

## 13. INTEGRATION

- godmode: the crew is part of the godmode protocol; the gate line reports the crew
  when fired.
- goal: the goal engine consumes the tasklist and the wave states.
- session-monitor-auto: arms on the crew; watches parent + children every beat.
- honestscore: scores the merged deliverable; the verify/adversarial seats feed it
  evidence.
- ideas: on kill criteria or 3-probe loops, ideas generates the alternative approach
  before a seat is re-dispatched.
- worktrees-single-writer: when build seats share a repo, this protocol defers to it
  for worktree ownership.

## 14. GATE-LINE CONTRIBUTION

When the crew runs, the parent's gate line reports:

```
subagents: fired (N seats, wave W, delivered D/killed K)
```

"fired" = real subagent launches this turn with tasklist rows backing them. A crew
announced but not launched (no tasklist, no session IDs) reports as "skipped: why" -
announcing a crew without launching it is the hollow-crew defect this skill exists
to prevent.

---

# APPENDIX A - SEAT-COUNT PLAYBOOK BY TASK SHAPE

| Task shape | Seats | Split |
|---|---|---|
| Trivial one-liner | 0 (solo) | crew skipped; say so honestly |
| Small bug fix, one file | 5 | recon, cross-check (does the bug exist as described), build, verify, adversarial |
| Feature across 3+ files | 6-9 | + domain cross-checks (tests, docs) |
| Multi-file refactor | 8-12 | + one build seat per worktree; regression verifier added |
| Research report | 6-10 | recon per source family; cross-check per claim cluster; synthesizer |
| Security audit | 8-15 | recon (surface), cross-check (CVE currency), 2-4 build (probes), 2 verify (FP rules), adversarial |
| Client website build | 8-15 | design recon, copy, build (split by page/section), a11y verify, visual judge, SEO cross-check |
| Fleet/harness verification | 6-10 | recon per host, cross-check spec-vs-live, adjudicator, independent re-scan, provenance verifier |
| Anything with legal/financial exposure | >= 8 | + one dedicated verify seat for numbers (recompute every aggregate) |

## A.1 The five-seat minimum, everywhere

Even a tiny task fills the floor honestly: recon (what exists), cross-check (is the
premise true), build (the change), verify (run it), adversarial (refute it). Five
small mandates beat one overgrown seat when the cost of a wrong answer is real.

---

# APPENDIX B - TASKLIST FILE SPEC

Path: `<run-dir>/tasklist.md` (created before wave 1; appended, never rewritten).

```markdown
# TASKLIST - <run slug>
Goal: <one line>
DoD: <numbered, measurable criteria>
Abort: <criteria>
Waves: <N>

| ID | Seat | Mandate | IN | OUT | Wave | Status | Evidence |
|----|------|---------|----|----|------|--------|----------|
| S1 | recon | map auth surface of /api | src/, docs/api.md | out/s1-map.md | 1 | delivered | out/s1-map.md (214 lines) |
| S2 | build | implement retry per S1 map | src/client.py | patch | 2 | delivered | commit 4f2c1a9 |
| S3 | verify | re-run pytest + race on merged tree | repo | out/s3-verify.md | 3 | running | - |
```

Status transitions: pending -> running (on launch) -> delivered (OUT exists AND
evidence column filled) | failed -> retry/reassign/absorbed | killed (reason logged).

A wave unlocks when every row it depends on is delivered or explicitly absorbed.
A row is never closed from chat alone - the Evidence column is the contract.

## B.1 The OUT file contract

Every seat writes its OUT as a file (chat responses are for beats, not for
evidence). Minimum shape:

```
# OUT - <seat id> - <one-line mandate echo>
FINDINGS: (bullets, each with evidence: command + output / file+bytes / HTTP status)
FILES: (touched/created paths with sizes)
ASSUMPTIONS: (explicit list, or "none")
UNVERIFIED: (what could not be measured + the exact command a verifier should run)
```

The orchestrator merges from OUT files. If an OUT is missing the EVIDENCE section,
the row stays UNVERIFIED and the next beat chases it.

---

# APPENDIX C - FAILURE SIGNATURE CATALOG (from real fleet runs)

These are the recurring seat-failure shapes; recognize them by signature, not by
trust:

- C1 Byte-identical outputs across seats: two seats returning the same fire list
  verbatim means one copied the other (or both copied a template). Dedupe by id and
  diff; treat identical lists as one unverified result, re-run one seat.
- C2 Phantom identifiers: a seat cites a tool/skill/file id that does not exist in
  the environment ("go-quality-gate" on a host that has "go-checker"). Any cited id
  that does not resolve locally invalidates the row that cited it.
- C3 Truncated-excerpt false negatives: the checker saw only a head-excerpt and
  reported absence of something that lives in the tail. Fix: re-check against the
  full artifact before believing any "absent" verdict.
- C4 Stored-flag drift: a seat's stored pass flag contradicted by an independent
  recomputation (17/20 mismatches in the jury round). The fresh recompute wins.
- C5 Duration/outcome nonsense: epoch-scale or 0.0 durations, outcome=null on
  pass=true rows. Impossible numbers mark the row UNRESOLVED, not pass.
- C6 The deadline false-fail: a seat killed by the wall-clock after doing real,
  evidenced work. Check the terminal outcome server-side before recording FAIL;
  record timeouts as timeout, distinct from quality failures.
- C7 The self-acknowledging correction: a seat responds to a watchdog correction
  with "acknowledged" and no action. Treat as no-response; re-inject with "execute
  immediately, no acknowledgment needed".

Each failure class maps to a kill/repair decision in Section 7-8; C1-C5 require
re-run (evidence-based), C6 requires outcome re-check before retry, C7 is an
escalation, not a retry.

---

# APPENDIX D - BACKGROUND VS FOREGROUND SEATS

- Foreground (default): the orchestrator waits. Use when the next step depends on
  the seat's OUT and nothing else can proceed.
- Background: use when seats are independent and long (research dives, slow builds);
  the monitor loop (session-monitor-auto) owns their beats. Do not poll busy-wait;
  arm the watchdog and continue.
- Mixing: wave launches may mix foreground and background only when the foreground
  seats do not depend on the background ones. If they do, split into waves (the
  wave rule already forbids this dependency inside one message).

---

# APPENDIX E - QUICK REFERENCE CARD

```
1. SCORE importance 0-10 -> crew size (floor 5 / ceiling 30)
2. WRITE tasklist.md (ID/Seat/Mandate/IN/OUT/Wave/Status/Evidence)
3. WAVE 1: recon + cross-check  (5-8 launches, one message)
4. MERGE wave-1 evidence; unlock wave 2 (build seats, disjoint files)
5. WAVE 3: verify seats re-run checks fresh
6. MERGE: evidence wins conflicts; one-writer merge
7. VERIFY-AFTER-MERGE: recompute headline claims from the merged artifact
8. ADVERSARIAL: fresh seat, mandate REFUTE, scope = stated requirements
9. CLOSE: every row delivered / killed / absorbed - with evidence
10. honestscore on the deliverable; report crew in the gate line
```

Floors and ceilings: 5 minimum, 30 maximum, 5-8 launches per message, depth 1,
retry <= 2, kill at 3 same-class probes, 0 tolerated silent drops.

---

# APPENDIX F - SEAT CONTRACT TABLE (IN / OUT per role)

| Role | Reads (IN) | Writes (OUT) | Done when |
|---|---|---|---|
| recon | code paths, docs, catalogs named in the tasklist | distilled map: files, interfaces, patterns to mirror, landmines | map exists, no raw dumps |
| cross-check | named claims + the sources to check against | per-claim verdict with citation or contradiction | every claim has a verdict |
| build | assigned file-set only | the changed artifact + build log | tooling passes; output file exists |
| verify | the merged artifact | check-by-check output with exit codes | every named check has fresh output |
| adversarial | the deliverable + stated requirements ONLY | refutation list or "no refutation survives" | each refutation carries evidence |
| domain specialist | its domain inputs | findings in its OUT with sources | mandate's success condition met |
| merge-prep | seat OUTs | deduped evidence bundle (one row per fact, source per row) | bundle builds from files alone |

## F.1 Mandate-writing guide

A mandate is ONE sentence containing: the object, the action, and the pass
condition.

- Bad: "research auth" (no pass condition, unbounded).
- Good: "enumerate every auth-touching route in src/api/ and return a table of
  route / auth check / gap, in out/s1-auth-map.md".
- Bad: "help with security" (no scope, no OUT).
- Good: "test the login rate limit: 5 attempts then 429 with Retry-After; paste the
  raw responses; mark UNVERIFIED anything you could not exercise."

If you cannot name the OUT path and its success condition, you are not ready to
launch the seat - finish the tasklist first.

---

# APPENDIX G - DEDUP: HOW TO PROVE SEATS DO NOT OVERLAP

Before launching a wave, for every pair of seats in it, answer: what does seat A
deliver that seat B does not? If you cannot answer in one sentence, merge the seats
or sharpen the mandates. Duplicated scope wastes a seat and poisons the merge (two
"verify everything" seats produce two half-ignored checklists).

Mechanical check after wave 1: list each seat's files-read / claims-verified; any
file or claim covered by two seats is either split cleanly (one owner per fact) or
one of the pair is reassigned. One owner per fact, like one writer per path.


