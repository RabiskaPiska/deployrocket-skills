---
name: auto-lossless-compaction
description: Lossless auto-compaction for long sessions. Tracks context fill; when fill hits window minus reserve (e.g. 1M window, compact at 850k = window-150k) it AUTOMATICALLY flushes EVERYTHING (decisions, facts-by-reference, secret pointers, open threads, file paths, in-flight task state, next actions) into the SQLite FTS5-indexed memory store via an 11-section handoff file, verifies zero missing items with a pre/post checklist diff, and only then allows compaction. Reinjects from the store at SessionStart so the continued session never says "I don't have X, provide API keys again". Fires always; costs ~0 tokens until the trigger line.
alwaysFire: true
triggers:
  - every session start (arm the tracker)
  - every substantive tool-use turn (read the fill gauge)
  - "compact", "context almost full", "before /clear", "handoff and continue"
  - any run where context-budget-manager reports fill >= 70% of window
---

# AUTO-LOSSLESS-COMPACTION SKILL (v1, spec section-02-compaction-skill)

## 0. THE ONE-SENTENCE CONTRACT

A session running this skill NEVER loses a decision, fact, secret-reference, open
thread, file path, or in-flight task state to context compaction - because the flush
protocol writes all of it to a durable, FTS5-indexed handoff store BEFORE the window
shrinks, and reinjection restores it on the other side.

Fire this skill ALWAYS (alwaysFire: true). Its steady-state cost is one integer
comparison per turn. It is allowed to fire on every session start and every
substantive tool-use turn without a relevance decision.

---

## PART A - WHY THIS EXISTS (the measured failure it kills)

On 1M-window fleet sessions (window raised to 900k on live locations, provider
trigger clamped to window-33000), long autonomous runs compact at roughly 850-900k
tokens. What the provider-native compaction keeps is a summarization decision outside
our control. Measured failure classes this skill kills:

1. "I don't have X, provide API keys again." The continued session re-asks the owner
   for credentials already supplied earlier in the session. Violates the
   no-confirmation directive and wastes a round trip on the one thing the owner must
   do by hand.
2. Decisions silently dropped. A settled "we chose option B because X" evaporates and
   the continued session re-derives (or contradicts) the decision - the exact
   confabulation class lesson-recall was built to stop.
3. In-flight task state lost. A background command or multi-wave subagent fleet is
   running when compaction fires; the continued session does not know the wave
   schedule, the ledger file, or the resume command.
4. File paths lost. Artifacts written to a VPS or to a Desktop backup folder are
   re-discovered by re-exploring instead of being read from a pointer.
5. Open threads dropped. "Task 7 of 9 waves failed with exit 2" survives only as a
   vague "some work remains", with no failure detail.
6. Facts-by-reference lost. "The client's DNS is hosted at registrar X, login held
   by owner" becomes unverifiable summary prose with no source pointer.

Why the provider's compaction alone is not enough: it compacts by summarizing
context held in RAM. It cannot write a durable file the NEXT session reads (a
different process), index the summary for FTS5 lookup, verify nothing was lost, or
protect secret REFERENCES from collapsing into indirections that no longer resolve.
Therefore the flush must happen BEFORE the provider compacts, into OUR store.

Design principle: SAVE-THEN-COMPACT, never compact-then-recover. Once fill crosses
the trigger line, the ONLY legal next action is the flush protocol. No new tool
work, no new subagent spawns, no long reads. The skill converts "context pressure"
into "a write deadline". After the flush verifies clean, compaction (ours or the
provider's) may destroy detail freely, because the store holds it.

What "nothing lost" does NOT mean: it does not mean the transcript is preserved
byte-for-byte. It means every item in the 11 content classes is present, pointered,
and queryable. Transcripts are the harness's business; decisions, facts, pointers,
threads, paths, and state are this skill's business.

Out of scope: fighting the provider's own compaction (we run BEFORE it and make our
flush the source of truth); changing the global autocompact window settings
(window clamped to model max, provider trigger window-33000 - we READ those settings,
never write them); redesigning the Obsidian brain (downstream consumer of the
handoff file, not the handoff store); secrets tooling beyond the flush-path
leak-gate (secrets-leak-hunter remains the canonical scanner).

---

## PART B - STATE AND VOCABULARY (glossary, use verbatim)

- WINDOW: the model's context window in tokens, as reported by
  context-budget-manager. Examples used throughout: 1,000,000 (1M), 900,000
  (fleet-clamped), 200,000 (small). The skill NEVER hardcodes a window; it reads
  the value at arm-time. If unreadable, assume the conservative fleet clamp of
  900,000 and log that assumption into the handoff header.
- RESERVE: the token budget kept for (a) the flush protocol's own writes, (b) the
  reinjection payload after compaction, (c) safety margin for provider-side
  accounting drift. Default 150,000 tokens; configurable via
  `compaction.reserve_tokens`, clamped to [30000, 300000].
  FORMULA: RESERVE = min(150000, max(30000, 0.15 * WINDOW)).
  Rationale for the small-window floor: on windows under 300,000 a 150k reserve is
  75% of the window and would trigger compaction absurdly early. RESERVE shrinks
  proportionally but never below 30,000.
- TRIGGER_LINE = WINDOW - RESERVE. Examples: 1M -> 850,000. 900k -> 765,000.
  200k -> 170,000. At TRIGGER_LINE the flush is MANDATORY.
- SOFT_LINE = 0.70 * WINDOW (config `compaction.soft_ratio`, clamp [0.5, 0.9]).
  At SOFT_LINE the skill starts requiring the ledger to be fresh but does not
  compact.
- FILL: current estimated context fill in tokens, taken from the
  context-budget-manager ledger (the single estimator; this skill never
  re-estimates fill itself).
- LEDGER: the per-session fill ledger maintained by context-budget-manager
  (cumulative token/cost tracking). THE LEDGER RULE: a flush may only fire if the
  ledger was updated within the last 3 turns (config
  `compaction.ledger_max_age_turns`); otherwise first force one cheap accounting
  pass so TRIP decisions use fresh numbers, THEN flush.
- HANDOFF_DIR: `<state-root>/handoffs/` where `<state-root>` is the memory store
  root (PC: `C:/Users/rabim/.config/opencode/memory/handoffs/`; VPSes:
  `/home/rabi/.config/opencode/memory/handoffs/`). Handoff files are named
  `handoff-YYYYMMDD-HHMMSS-<slug>.md` and are APPEND-ONLY once written (never
  edited in place; corrections go in an appended ADDENDUM block).
- STORE: the fleet SQLite memory DB (path from session-memory-auto config; default
  `~/.config/opencode/memory/memory.db`). Holds the FTS5 virtual table
  `memory_fts` plus the metadata table `handoff_meta`. The store is the
  machine-queryable index; the handoff file is the human-readable artifact. BOTH
  are written; BOTH are verified.
- IN-FLIGHT REGISTRY: the skill's own JSON file `compaction-state.json` in
  HANDOFF_DIR, listing background jobs / long commands the session started
  (pid, command, log path, started-at, owner task, keep-or-kill verdict).
  Written at arm-time and updated on every background spawn. The flush protocol
  reads it to capture in-flight state. A background job with no registry entry,
  discovered during the flush, is registered NOW (late registration) rather
  than skipped - an unregistered in-flight job post-compact is a spec violation
  of the spawn discipline, logged as a lesson.
- CHECKLIST: the 11-item loss checklist (Part F). Each item is PRESENT (the
  handoff file contains a non-empty section with at least one concrete entry)
  or MISSING. The flush gate: zero MISSING.
- REINJECT_BUDGET: max tokens of reinjection payload at SessionStart.
  Default 12,000; floor 4,000.

---

## PART C - TRIGGER MATH (the exact algorithm, normative)

### C1. ARM leg (session start)

1. Read WINDOW from the model/provider metadata via context-budget-manager's
   accounting. Do not hardcode. If unreadable, assume the conservative fleet
   clamp of 900,000 and log the assumption into the handoff header.
2. Compute RESERVE = min(150000, max(30000, 0.15 * WINDOW)).
3. Compute TRIGGER_LINE = WINDOW - RESERVE and SOFT_LINE = 0.70 * WINDOW.
4. Create/refresh `compaction-state.json` with: window, reserve, trigger_line,
   soft_line, session_id, started_at, in_flight: [], last_reinjected_sha: null.
5. Log one line: `auto-lossless-compaction ARMED window=<W> trigger=<T> soft=<S>`.
6. Register the SessionStart reinjection hook (Part E) if not already installed.
7. Run the idempotent store DDL (Part H) against the store.

Config validation at arm time: reserve in clamp range, soft_ratio in [0.5, 0.9],
reinject_budget >= 4000. Invalid config = REFUSE TO ARM and warn loudly; a
half-configured lossless guard is worse than an absent one because it breeds
false confidence.

Config is read at ARM time only; changing config mid-session takes effect next
session, never mid-flight, so TRIP decisions are stable for a session's lifetime.

### C2. The trigger table (values are the CONTRACT)

| WINDOW    | RESERVE | SOFT (70%) | TRIGGER (window - reserve) |
|-----------|---------|------------|----------------------------|
| 1,000,000 | 150,000 | 700,000    | 850,000                    |
| 900,000   | 135,000 | 630,000    | 765,000                    |
| 400,000   | 60,000  | 280,000    | 340,000                    |
| 300,000   | 45,000  | 210,000    | 255,000                    |
| 200,000   | 30,000  | 140,000    | 170,000                    |

Worked check of the formula, row 1: 0.15 * 1,000,000 = 150,000;
min(150000, 150000) = 150,000; 1,000,000 - 150,000 = 850,000. Correct.
Worked check, row 5: 0.15 * 200,000 = 30,000; the floor keeps it at 30,000;
200,000 - 30,000 = 170,000. Correct.
Worked check, row 2: 0.15 * 900,000 = 135,000; min(150000, 135000) = 135,000;
900,000 - 135,000 = 765,000. Correct.

### C3. Owner-band rule

When WINDOW = 1,000,000, the skill MUST trip no later than 900,000 and SHOULD
trip at 850,000. Acceptance rule: any observed trip ABOVE 900,000 on a 1M window
is a spec violation and fails the install test.

### C4. Per-turn gauge (the FIRE leg, every substantive tool-use turn)

1. Read FILL from the context-budget-manager ledger (last update). If the ledger
   is stale (> 3 turns), update it first (Ledger rule).
2. If FILL >= TRIGGER_LINE: enter FLUSH-MANDATORY state. Immediately:
   a. Stop initiating new tool work (new spawns, long reads, builds).
   b. Run the pre-compact flush protocol (Part D) top to bottom.
   c. Run the verification gate (Part F). If it fails, repair and re-verify.
   d. Only then proceed to compaction: if the provider compacts automatically at
      its own threshold, let it - the store now holds everything. If the provider
      does not, emit the explicit compaction signal (/compact or harness
      equivalent) and write to the session log:
      `auto-lossless-compaction FLUSH OK fill=<F> handoff=<file> store_rows=<N>`.
3. Else if FILL >= SOFT_LINE: emit a one-line warning into the session transcript:
   `auto-lossless-compaction SOFT fill=<F>/<W> (>=70%). Ledger must stay fresh;
   finish or checkpoint long-running work now.` and enforce the Ledger rule.
4. Else: no-op. The arm-time hook already did its work; the cost of this leg is
   one integer comparison.

### C5. Determinism requirement

The comparison uses >= on integers, never "about" or "roughly". A measured FILL
of 849,999 on a 1M window does NOT trip; 850,000 does. The red test depends on
this exactness.

### C6. Why window-150k and not the fleet's window-33000 provider trigger

The fleet clamp (window-33000) is the PROVIDER-side emergency trigger. Ours is the
AGENT-side orderly trigger, roughly 4x earlier, because the flush protocol needs
headroom to write the handoff (typically 8-20k tokens of tool I/O) while the
session is still fully coherent. Compacting with only 33k of headroom means
writing the most important artifact of the session under memory pressure - the
exact condition that produces loss.

### C7. Degraded mode (no ledger)

If context-budget-manager is absent, the skill arms in DEGRADED mode: it assumes
fill grows monotonically and trips at the harness's own compaction-warning signal
instead, logging `DEGRADED: no ledger` in the handoff header. A missing estimator
must never become a silent no-guard.

---

## PART D - THE PRE-COMPACT FLUSH PROTOCOL (11 sections, F1..F8, step-by-step)

This is the core. The protocol writes ONE handoff file with EXACTLY these 11
sections, in this order, then indexes it. Nothing skipped; a section with
genuinely nothing to report still gets its header plus the literal line
`NONE (verified: nothing to record)`.

### F1. Freeze intake

Announce in one line: `auto-lossless-compaction FLUSH START fill=<F> trigger=<T>`.
From this line on: no new spawns, no new background jobs, no long reads.
Already-running background jobs are CAPTURED, not stopped (see F6 and S08).

### F2. Gather raw state (5 cheap reads, all local, all BEFORE any summarization)

a. The session's own transcript tail (last ~200 turns, or what the harness
   exposes of it).
b. `compaction-state.json` (the in-flight registry).
c. The context-budget-manager ledger (fill history, cost, burn rate).
d. Any TASKLIST / NOTES / CONTINUE-HERE files this session wrote or read.
e. The session-memory-auto observation log for this session (already-compressed
   tool I/O observations - REUSE them, do not re-summarize raw bytes; no
   double-summarization drift: quote observation entries verbatim where they
   exist).

### F3. Write the handoff file

Write to HANDOFF_DIR with the naming rule from Part B. Write it INCREMENTALLY
(open in append mode, write section by section) so a crash mid-flush leaves a
prefix on disk, not nothing. EXACTLY these 11 sections, in this order. Each
section below lists its REQUIRED content and its completeness rule. A section
with genuinely nothing to report still gets its header plus
`NONE (verified: nothing to record)`.

SECTION 1 - SESSION IDENTITY AND WINDOW MATH
  session id, host (pc / vps-webdev-179.61.138.8 / vps-knowledge-179.61.251.98),
  model, started_at, window, reserve, trigger_line, fill at flush time,
  compact_due reason (e.g. "fill 851000 >= trigger 850000").
  Completeness rule: all 9 fields non-empty.

SECTION 2 - TASK AND GOAL STATE
  The declared goal(s), the current wave/phase, the compound stop condition,
  progress count (done / in-flight / remaining). Copy the exact goal text; do
  not paraphrase a goal - paraphrase is where "keep going until X" becomes
  "some X-ish thing".
  Completeness rule: verbatim goal text present + a progress count.

SECTION 3 - DECISIONS MADE AND WHY
  Every load-bearing decision with: decision, alternatives rejected, the why,
  the evidence pointer (tool result / file / URL). Include what was REJECTED -
  the continued session's #1 rediscovery cost is re-proposing rejected options.
  Completeness rule: >= 1 decision with a why + a rejected alternative, or the
  NONE line with the verified-empty note.

SECTION 4 - FACTS LEARNED (BY REFERENCE)
  New facts established this session, each with: the fact, the source (URL, file
  path, command + output hash, or doc section), the date verified. Facts WITHOUT
  a source pointer go in marked UNVERIFIED, never silently.
  Completeness rule: every fact line contains a `src:` pointer.

SECTION 5 - SECRETS AS POINTERS (NEVER VALUES)
  For every credential the session used or was given: the POINTER only - env var
  name, vault path, config file + JSON path, keychain entry, service + owner
  contact. ABSOLUTE BAN on secret values in the handoff. Before the file is
  indexed (F5), the leak-gate (failure mode G3) scans Section 5 for value-shaped
  strings (long base64/hex, sk- prefixes, PEM bodies, bearer tokens) and FAILS
  the flush if any match. The continued session reads the pointer, not the
  value; if the value was in old context and is gone, the session says "secret
  available at <pointer>, re-read from there" - never "provide it again" to a
  human when the pointer suffices, and never invents the value.
  Completeness rule: every secret line is `name -> pointer`, zero value-shaped
  strings (gate-enforced).

SECTION 6 - FILE PATHS AND ARTIFACTS
  Every file created/modified/deleted this session: absolute path, one-line
  description, created-or-modified marker, and (for build artifacts) the
  verification state (built? tested? untested?). Include scratch files and the
  exact paths of any CONTINUE-HERE / handoff predecessors.
  Completeness rule: >= 1 path or the verified NONE line; every path absolute.

SECTION 7 - OPEN THREADS
  Unresolved questions, blocked items, half-finished analyses, and for each: why
  open, what unblocks it, who/what owns it (me / owner / external service /
  another session). Threads awaiting ONLY owner input (API key, login, payment)
  are flagged `[OWNER-BLOCKED]` with the single named item (the fleet
  three-things rule: credential, registration, payment).
  Completeness rule: every thread has a status + an unblock condition.

SECTION 8 - IN-FLIGHT COMMANDS AND PROCESSES
  Copied from the in-flight registry: pid, host, command line, log path,
  started_at, expected side-effect, and how to check it is alive (the EXACT
  probe command). For each, state keep-or-kill policy: (a) SURVIVES compaction -
  detached, log path recorded, next session polls the log; (b) MUST DIE - kill
  command recorded. If none: NONE line. A background job with no registry entry,
  discovered here, is registered NOW (late registration) rather than skipped.
  Completeness rule: every in-flight entry has a probe command; every entry has
  a keep-or-kill verdict.

SECTION 9 - NEXT ACTIONS (ORDERED)
  The exact continuation plan: ordered steps, each with its command or edit
  target and its pass/fail check. This is the section a fresh session executes
  FIRST. It must be executable without asking the user anything that a file or
  store query could answer.
  Completeness rule: ordered list, every step has an action + a check.

SECTION 10 - FAILURES TRIED AND RULES LEARNED
  Failed approaches with symptom -> cause -> fix, in lesson-recall format, so
  the continued session (and future spawns, via lesson-recall) gets them
  verbatim. Include any gate that fired and why.
  Completeness rule: >= 1 lesson or verified NONE.

SECTION 11 - VERIFICATION EVIDENCE AND HANDOFF CHECKLIST
  The pre/post diff (Part F) result: the 11-item checklist with PRESENT/MISSING
  per item, the FTS5 row count delta, the sha256 of this file at close, and the
  flush duration. Written LAST; this is what makes the handoff auditable by a
  fresh process.
  Completeness rule: checklist present, 11/11 PRESENT, sha256 recorded.

### F4. Close and hash the file

Compute sha256 of the file body. Append the final line:
`FLUSH-CLOSED sha256=<hash> sections=11/11 fill_before=<F>`.
(The checklist in Section 11 is what makes "11/11" a claim with evidence, not a
self-grade.)

### F5. Index into the FTS5 store (exact SQL in Part H of the body)

One row per section (11 rows minimum) plus one row for the whole-file summary,
each tagged with session_id, host, created_at, section name, and kind
(`handoff`). The FTS5 row count delta is what Section 11 records. The Section 5
body is scanned (G3) BEFORE the INSERT runs; a failed scan means the section
body is re-written with the offending value replaced by its pointer, re-scanned,
THEN inserted. The DB never receives a value-shaped string from the flush path.
Wrap the index writes in a transaction (failure mode G8).

### F6. In-flight job handoff

For every SURVIVES entry in Section 8, verify RIGHT NOW that the log path exists
and is growing (one stat/read). If a job's log is NOT growing, downgrade it to a
Section 7 open thread with the stall noted - do not hand a dead job to the next
session as if it were alive.

### F7. Reinjection readiness

Confirm the SessionStart reinjection (Part E) will find this handoff: query the
store for the newest handoff rows for this session_id and record the top hit in
the session log.

### F8. Announce completion and ALLOW compaction

`auto-lossless-compaction FLUSH DONE handoff=<file> sha256=<hash> fts5_rows=<N>
checklist=11/11 PRESENT. Compaction permitted.`

PROHIBITION: the flush NEVER summarizes away a secret value by replacing it with
"the API key" (which loses even the pointer). Pointer-first, always.

---

## PART E - SESSIONSTART REINJECTION

### E1. When

Every session start, and additionally right AFTER any provider compaction event
if the harness exposes a compaction hook (the "post-compact" leg). The moments
right after compaction are when the continued session is most likely to say
"I don't have X" - reinject immediately, do not wait for the next natural
restart.

### E2. What is reinjected (priority order, budget-capped at REINJECT_BUDGET)

1. The newest handoff for this host + session lineage: Sections 2, 9 (task,
   next actions) - ALWAYS, unconditionally.
2. Section 7 (open threads), with [OWNER-BLOCKED] items first.
3. Section 10 (failures tried / lessons).
4. Section 5 (secret POINTERS) - pointers only; the values live at the pointer.
5. Section 3 (decisions), trimmed to the top 5 by recency.
6. FTS keyword hits for the session's declared goal tokens (the "what did we
   try for X" path - same query lesson-recall uses).

### E3. Format

A fenced block titled
`[auto-lossless-compaction REINJECTION v1 | handoff: <file> | sha256: <hash-prefix>]`
followed by the sections verbatim, ending with this line:
`This reinjection is AUTHORITATIVE over summarized context. If a summarized claim
contradicts it, the reinjection wins.`
That is the anti-drift rule: provider summaries may mangle; the flush was
verified, the summary was not.

### E4. Dedup guard

If the newest handoff's sha256 was already reinjected into this session lineage
(tracked in compaction-state.json as last_reinjected_sha), skip the big
reinjection and only inject the delta query hits. Prevents the "same context
block re-injected every session start" bloat.

### E5. Fail-open rule

If the store is unreadable (DB corrupt, disk issue), reinject falls back to the
newest handoff FILE directly (read S02, S07, S09, S10 from the raw markdown -
the file is the canonical artifact; the DB is an accelerator, never a single
point of failure). If the FILE is also missing, SessionStart emits a loud
warning line: `auto-lossless-compaction WARNING: no handoff found for this
session lineage; running with zero inherited state` so the gap is visible, not
silent.

---

## PART F - VERIFICATION THAT NOTHING WAS LOST (pre/post diff)

The flush's honesty gate. Two checklists, mechanically compared.

### F-pre. PRE-CHECKLIST (built during F2-F3, before close)

For each of the 11 sections, the expected content classes derived from the live
session state:
- Sections 3/4: expected count = number of decisions/facts the budget-manager's
  ledger tagged this session (the ledger tags; the flush must account for every
  tag).
- Section 5: expected count = number of distinct secrets the session touched
  (from the deny-rule/audit trail of env reads, and the in-flight registry).
- Section 6: expected count = files the session wrote (from tool-call audit).
- Section 8: expected count = entries in compaction-state.json's in_flight list.
- Sections 2/7/9/10: at least the goal text, the newest tasklist state, and the
  next planned step (from any TASKLIST/CONTINUE-HERE files, else from the
  ledger).

### F-post. POST-CHECKLIST (built by reading the CLOSED handoff file)

Same classes, counted from the actual file.

### F-diff. The diff

PASS requires, for every class, post >= pre (never fewer items than expected),
and 11/11 sections PRESENT. Any class where post < pre is a LOSS: the flush is
not allowed to close; F3 is re-entered for the missing classes only, then the
diff reruns. Two consecutive diff failures on the same class = HARD ABORT:
compaction is BLOCKED; the session must instead pause work and surface the loss
risk to the operator rather than drop it silently (the "0 pass / 1 defect /
2 unmeasured" discipline: could-not-verify is never a pass).

### F-12. The zero-missing-items proof standard

A session plants 12 known items (3 decisions, 3 facts with sources, 2 secret
pointers, 2 file paths, 1 open thread, 1 in-flight job, next actions). The post
checklist must account for all 12, and the FTS5 query must retrieve >= 11 of
them by keyword (>= 92% recall; the 1 allowed miss must be explained in the
test report - typically a stopword-collision artifact, and the test asserts
WHICH one and why).

### F-not. What "nothing lost" does NOT mean

It does not mean the transcript is preserved byte-for-byte. It means every item
in the 11 content classes is present, pointered, and queryable. Transcripts are
the harness's business; decisions, facts, pointers, threads, paths, and state
are this skill's business.

---

## PART G - FAILURE MODES AND MANDATORY HANDLING (8 modes)

### G1. Compaction mid-command (the provider compacts while a long command runs)

The in-flight registry exists precisely for this. The command was registered at
spawn (pid, log, expected side-effect). Post-compact, the FIRST SessionStart
action is to poll every SURVIVES job's log (F6 logic) and reinject "job X
running since T, log at P, next check command C". Mandatory pre-registration is
what makes this a non-event. A job found unregistered post-compact is a spec
violation of the spawn discipline, logged as a lesson (S10).

### G2. Crash during flush (host dies between F3 and F4)

Incremental append (F3) means the file exists as a prefix without a
FLUSH-CLOSED line. SessionStart (next boot) scans HANDOFF_DIR for files lacking
FLUSH-CLOSED; each is treated as an open handoff: the reinjection reads it
anyway (prefix is truth, marked PARTIAL), and the session's first duty is to
complete the missing sections from the live store and observations, then close
it properly. A prefix is NEVER treated as garbage.

### G3. Secret leak into the handoff

The value-shape scan (long hex/base64 runs, `sk-`, `Bearer `, PEM headers,
`AKIA`-style key prefixes, high-entropy quoted strings >= 20 chars) runs on the
file BEFORE F5/F7. On hit: FAIL the close, replace value with pointer, re-scan.
The gate runs even on manual edits because F4's close re-checks.
secrets-leak-hunter remains the deeper audit tool; this gate is the flush-path
tripwire only. If a value ever crosses hosts, STOP: rotate the secret
(secret-rotation skill), fix the gate, re-run.

### G4. Stale ledger (fill number wrong)

The Ledger rule (Part B). If TRIP fires on a stale ledger and the true fill is
later found > TRIP + 10%, the flush still proceeds (over-flushing is cheap;
under-flushing is the disaster). Never delay a flush because "the number might
be wrong".

### G5. FTS5 unavailable (sqlite built without fts5 on minimal images)

Detected at arm-time with a fts5 probe (`SELECT fts5('test')` or
`SELECT sqlite_compileoption_used('ENABLE_FTS5')`); on failure, the store falls
back to a LIKE-scan table with the same columns and a loud one-line warning in
every reinjection: `FTS5 UNAVAILABLE - LIKE fallback active, retrieval is
substring only`. Install verification checks FTS5 presence precisely to avoid
shipping this fallback silently.

### G6. Duplicate compaction loops (flush fires, provider compacts, reinjection
bloats context, fill re-trips)

The dedup guard (E4) + reinjection budget (12,000 tokens) + a hard rule: the
flush does not count its own handoff writes toward the FILL that trips the NEXT
flush in the same session lineage (fill is measured pre-flush). A second trip
within the same lineage requires a NEW 70% of fresh work, not reinjection mass.

### G7. Clock skew across hosts (git-synced handoff files compare created_at)

created_at is UTC ISO-8601 always; the lag check (Part H, H-lag) uses
sha256 membership, not timestamps, as the source of truth for "is this indexed".
Timestamps are for humans.

### G8. Store write fails mid-index (disk full, read-only mount)

F5 wraps the index writes in a transaction; on failure the flush is NOT DONE -
the file may close (F4) but F8 is never printed; the session logs
`FLUSH INCOMPLETE: store unwritable (<err>)` and must not allow compaction. It
may continue working only if it keeps the handoff file as the authoritative
record and retries indexing once per SessionStart.

---

## PART H - THE SQLITE FTS5 INDEX WRITE (exact schema and queries)

### H1. Store location

`~/.config/opencode/memory/memory.db` (PC and both VPSes use their own local
store; the fleet's git-sync layer propagates handoff FILES, and each host
re-indexes its own DB at SessionStart if the DB lags the files - see H-lag).

### H-schema. Schema (idempotent DDL, run at arm-time)

```sql
CREATE TABLE IF NOT EXISTS handoff_meta (
  id INTEGER PRIMARY KEY,
  session_id TEXT NOT NULL,
  host TEXT NOT NULL,
  section TEXT NOT NULL,          -- 'S01'..'S11' or 'WHOLE'
  kind TEXT NOT NULL,             -- 'handoff'
  file_path TEXT NOT NULL,
  sha256 TEXT NOT NULL,
  created_at TEXT NOT NULL,       -- ISO-8601 UTC
  body TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_handoff_meta_session ON handoff_meta(session_id);
CREATE INDEX IF NOT EXISTS idx_handoff_meta_created ON handoff_meta(created_at);

CREATE VIRTUAL TABLE IF NOT EXISTS memory_fts USING fts5(
  body,
  session_id UNINDEXED,
  host UNINDEXED,
  section UNINDEXED,
  kind UNINDEXED,
  file_path UNINDEXED,
  created_at UNINDEXED,
  tokenize = 'porter unicode61'
);
```

### H-insert. Insert (one call per section; the WHOLE row holds a <= 2000-char
digest plus the file path so FTS hits can redirect to the full file)

```sql
INSERT INTO memory_fts (body, session_id, host, section, kind, file_path, created_at)
VALUES (?, ?, ?, ?, 'handoff', ?, ?);
INSERT INTO handoff_meta (session_id, host, section, kind, file_path, sha256,
                          created_at, body)
VALUES (?, ?, ?, 'handoff', ?, ?, ?, ?);
```

### H-losscheck. The loss-check query (used by the verification gate)

```sql
SELECT section, COUNT(*) FROM handoff_meta
WHERE session_id = ? AND kind = 'handoff'
GROUP BY section;
```

Gate: rows for S01..S11 must all be >= 1. Any missing section = flush failure.

### H-reinject. The reinjection query (used by SessionStart)

```sql
SELECT body, file_path, created_at FROM memory_fts
WHERE memory_fts MATCH ?
  AND kind = 'handoff'
ORDER BY created_at DESC
LIMIT ?;
```

With the caller's task keywords as the MATCH argument (the same query
session-memory-auto's FTS5 lookup uses - one corpus, one query path).

### H-lag. File-lag repair

Because handoff FILES travel via the fleet git-sync layer while the SQLite DB
is per-host, SessionStart compares `SELECT MAX(created_at)` in handoff_meta
against the newest handoff file mtime under HANDOFF_DIR; if the DB lags, it
re-indexes any unindexed files (match on sha256 in handoff_meta) before serving
queries. This keeps "which host has the newest memory" a non-question: every
host self-heals its index from the canonical files.

### H-secretgate. Secret gate before indexing

The Section 5 body is scanned (G3) BEFORE the INSERT runs. A failed scan means
the section body is re-written with the offending value replaced by its
pointer, then re-scanned, THEN inserted. The DB never receives a value-shaped
string from the flush path.

---

## PART I - INTEGRATION POINTS (4 sibling skills)

### I1. context-budget-manager (fill accounting)

- Relationship: context-budget-manager is the LEDGER - it estimates context
  consumption each turn, logs the cumulative ledger, warns at 40-50% fill. It
  is the single source of FILL numbers. auto-lossless-compaction consumes its
  output and adds the DECISION layer: it is the component that decides that a
  number means "stop everything and flush".
- Contract: compaction reads `fill_tokens`, `window_tokens`,
  `ledger_updated_at` from the budget-manager's ledger; it never re-estimates
  fill itself (one estimator, one truth). If the budget-manager is absent,
  compaction arms in DEGRADED mode: it assumes fill grows monotonically and
  trips at the harness's own compaction-warning signal instead, logging
  `DEGRADED: no ledger` in the handoff header.
- Division of labor: budget-manager warns; compaction ACTS. The two must agree
  on RESERVE: budget-manager's compaction threshold minus compaction's reserve
  are the same number by construction (both read the same config key
  `compaction.reserve_tokens`).

### I2. session-memory-auto (capture leg)

- Relationship: session-memory-auto is the CONTINUOUS capture leg - hooks
  capture tool I/O each substantive turn and compress into observations,
  biased toward decisions and rules. auto-lossless-compaction is the BULK flush
  at the end. Complementary, not redundant: capture is lossy-compressed and
  continuous; flush is complete-and-verified and fires once at the line.
- Contract: the flush protocol REUSES session-memory-auto's observation log
  (F2e) as its primary raw material, and writes into the SAME store schema
  (Part H), tagged kind='handoff' so the corpus stays one query surface. The
  flush must NOT re-compress what observations already compressed (no
  double-summarization drift: the file quotes the observation log entries
  verbatim where they exist).

### I3. context-handoff (the 11-section format)

- Relationship: context-handoff OWNS the 11-section handoff format.
  auto-lossless-compaction is its AUTOMATION: it fires context-handoff's
  format mechanically at the trigger line instead of waiting for a human or a
  model to decide a handoff is due.
- Contract: section names, order, and completeness rules in Part E (F3) are
  the single format spec; context-handoff and this skill reference the same
  numbered sections. Any format change lands in context-handoff first, then
  here, with the frozen-spec hash updated in both. The pre/post diff (Part F)
  is the enforcement context-handoff lacked: it documented the format; this
  skill makes skipping a section a hard failure.

### I4. lesson-recall (re-injection at spawns)

- Relationship: lesson-recall mechanically injects top-N store lessons into
  every spawn brief. The flush's Section 10 (failures tried) writes in exactly
  that lesson format, so the fleet's next spawns get the session's hard-won
  lessons even though the session that learned them was compacted away.
- Contract: flush-written lessons are tagged kind='handoff' + source_session;
  lesson-recall's query is extended to include them (no schema fork).
  Post-compaction spawn briefs therefore carry pre-compaction lessons: the
  confabulation guard survives compaction.

### I5. skill-frozen-spec (spec drift guard)

The source spec (docs/v66-sections/section-02-compaction-skill.md) is THE spec.
At install time the executor computes the spec hash and stores it with the
skill. Any later edit to the trigger math, the 11 sections, or the diff gate
must re-lock the hash - the same discipline every fleet spec skill uses.

---

## PART J - CONFIGURATION REFERENCE

All keys live in the skill's config block (per-host, seeded at install; the
skill reads config at ARM time only):

| Key                      | Default      | Meaning                                        |
|--------------------------|--------------|------------------------------------------------|
| compaction.reserve_tokens| (computed)   | RESERVE; if set explicitly, overrides the min/max formula but clamped to [30000, 300000] |
| compaction.soft_ratio    | 0.70         | SOFT_LINE multiplier on WINDOW                 |
| compaction.reinject_budget | 12000      | max tokens of reinjection payload at SessionStart |
| compaction.ledger_max_age_turns | 3     | ledger staleness bound before a forced update  |
| compaction.handoff_dir   | (per-host)   | HANDOFF_DIR override                           |
| compaction.store_path    | (per-host)   | memory.db path override                        |
| compaction.secret_patterns | (built-in) | additional value-shape regexes for G3         |
| compaction.trip_log      | (session log)| where trip/flush lines are written            |

Validation at arm time: reserve in clamp range, soft_ratio in [0.5, 0.9],
reinject_budget >= 4000. Invalid config = refuse to arm and warn loudly.

---

## PART K - PER-HOST DEPLOYMENT TABLE (paths pinned at install)

| Host                       | Skills dir                                                    | State root (HANDOFF_DIR parent)                      | Store (memory.db)                          |
|----------------------------|---------------------------------------------------------------|------------------------------------------------------|--------------------------------------------|
| PC (Windows)               | C:/Users/rabim/.config/opencode/skills/auto-lossless-compaction/ | C:/Users/rabim/.config/opencode/memory/             | C:/Users/rabim/.config/opencode/memory/memory.db |
| VPS web-dev 179.61.138.8   | /home/rabi/.config/opencode/skills/auto-lossless-compaction/  | /home/rabi/.config/opencode/memory/                  | /home/rabi/.config/opencode/memory/memory.db |
| VPS knowledge 179.61.251.98| /home/rabi/.config/opencode/skills/auto-lossless-compaction/  | /home/rabi/.config/opencode/memory/                  | /home/rabi/.config/opencode/memory/memory.db |

Notes:
- On Windows the sqlite3 CLI may not be on PATH; the skill uses Python's stdlib
  `sqlite3` module for both the DDL and the runtime queries. All three hosts
  standardize on Python so probe scripts are byte-identical across hosts.
- HANDOFF_DIR files sync via the fleet git layer (one repo, three brains, three
  hosts). The DBs do NOT sync; each host re-indexes synced files at SessionStart
  per H-lag. The cross-host drill verifies this self-heal explicitly.
- The PC's state root is inside the auto-memory dir, which the Stop-hook backup
  already mirrors to OneDrive - so even a dead PC disk retains the last handoff.

---

## PART L - CONTROL-FLOW PSEUDOCODE (normative)

This pseudocode is the executable skeleton the install must reproduce; comments
mark the spec-section each block implements.

```
state ARM(session, window):                      # C1
    W = window_from_provider(session)            # never hardcoded
    R = min(150000, max(30000, 0.15 * W))
    TRIGGER = W - R
    SOFT = 0.70 * W
    write compaction-state.json {W, R, TRIGGER, SOFT, session_id, started_at,
                                 in_flight: [], last_reinjected_sha: null}
    ensure_store_schema()                        # Part H, idempotent
    log "auto-lossless-compaction ARMED window=<W> trigger=<TRIGGER> soft=<SOFT>"

gauge per_turn():                                # C4
    F = ledger_fill()                            # fresh per Ledger rule
    if F >= TRIGGER: flush_mandatory()           # Part E F1..F8 + Part F diff
    elif F >= SOFT: log soft warning             # no compaction

flush_mandatory():                               # Part E
    announce FLUSH START; freeze intake          # F1
    raw = gather(transcript_tail, registry, ledger, tasklists, observations)  # F2
    file = HANDOFF_DIR / name(session)           # F3, incremental append
    for S in [S01..S11]: write_section(file, S)  # F3 completeness rules inline
    sha = close_file(file)                       # F4, FLUSH-CLOSED line
    secret_gate(file)                            # G3 scan BEFORE index
    index_into_fts5(file)                        # F5 -> H-insert, transactional
    verify_inflight_logs_growing()               # F6, downgrade dead jobs
    confirm_reinjection_ready(session)           # F7
    diff = pre_post_checklist(raw, file)         # Part F
    if diff != PASS: repair(missing); goto flush_mandatory (max 2 retries)
    announce FLUSH DONE; compaction_permitted()  # F8

session_start():                                 # Part E
    repair_file_lag()                            # H-lag, DB vs files
    h = newest_handoff_for(session_lineage)      # file fallback per E5
    if h.sha == last_reinjected_sha: inject_delta_only()   # dedup, E4
    else: inject(S02, S09, S07, S10, S05, S03_top5, fts_hits(goal_tokens))
          within reinject_budget                                  # E2
    mark last_reinjected_sha = h.sha
    poll all in_flight SURVIVES jobs             # G1 post-compact leg
```

The pseudocode is the contract for the simulator (work/x-compact/sim_fill.py):
the simulator implements exactly these blocks, so the green proof exercises the
real decision points, not a parallel approximation.

---

## PART M - FAILURE MODES: THE 8 MODES WITH MANDATORY HANDLING (summary table)

| #  | Failure mode                          | Mandatory handling (reference)                          |
|----|---------------------------------------|---------------------------------------------------------|
| M1 | Compaction mid-command                | In-flight registry pre-registration; post-compact first action polls every SURVIVES job's log (G1) |
| M2 | Crash during flush                    | Incremental append; prefix treated as truth marked PARTIAL; resumed session completes and closes it (G2) |
| M3 | Secret leak into handoff              | Value-shape scan BEFORE index; fail the close, replace with pointer, re-scan (G3) |
| M4 | Stale ledger                          | Ledger rule: force update if > 3 turns; over-flush rather than delay (G4) |
| M5 | FTS5 unavailable                      | Arm-time probe; explicit LIKE fallback + loud warning line in every reinjection (G5) |
| M6 | Duplicate compaction loops            | Dedup guard + 12k reinject budget + fill measured pre-flush only (G6) |
| M7 | Clock skew across hosts               | UTC ISO-8601 always; sha256 membership is the index-truth, timestamps for humans (H-lag, G7) |
| M8 | Store write fails mid-index           | Transactional index; file may close but F8 never prints; log FLUSH INCOMPLETE; retry once per SessionStart (G8) |

---

## PART N - WORKED EXAMPLES

### N1. The 1M-window overnight build

Context: a 10-wave site build on the web-dev VPS, model window 1,000,000. Hours
in, context-budget-manager's ledger reads FILL = 852,400. TRIGGER_LINE = 850,000.
852,400 >= 850,000, so TRIP.
The skill: freezes intake (no wave 8 spawns), reads the in-flight registry (one
background `npm run build` watcher, pid 4411, log
`/srv/dr/projects/site-a/build.log`), and writes the handoff:
- S02: goal verbatim "site-a build to >=9/10 all pages, all viewports"; progress
  7/10 waves done, wave 7 verification pending.
- S03: decisions - "Tailwind not CSS modules (client's other 2 sites use it,
  maintenance symmetry)"; rejected "MUI (bundle size 3x, client rejected)".
- S04: facts - "client's staging host is 179.61.138.8:8443 (src: client email
  2026-11-12, file /srv/dr/projects/site-a/NOTES.md)".
- S05: secrets - "DEPLOY_SSH_KEY -> /home/rabi/.ssh/id_ed25519 (vault ref only)";
  zero values.
- S06: 14 file paths incl. the pending component `src/components/Pricing.tsx`
  (written, untested).
- S07: open thread - "lighthouse CI badge pending owner's GitHub app install
  [OWNER-BLOCKED: GitHub App install]".
- S08: the watcher, probe `kill -0 4411 && tail -n 5 /srv/dr/projects/site-a/build.log`,
  verdict SURVIVES.
- S09: next actions - "(1) run axe+pa11y on /pricing, (2) fix any contrast <
  4.5:1, (3) re-run W1 visual QA, each with its pass check".
- S10: "contrast checker false-positived on gradient text; cause: axe sees
  pseudo elements; fix: tested real element styles directly".
- S11: checklist 11/11, sha256 recorded, FTS5 rows = 12.
Compaction happens. Session continues. When it asks "where is the deploy key?"
the reinjected S05 pointer answers without bothering the owner. Owner never
re-supplies a credential. That is the owner's exact requirement, met.

### N2. Compaction mid-wave with a background job

Fill trips at 765,300 on a 900k-clamped window mid-wave-4; a long `pytest -q` is
running (pid 8820). S08 registers it SURVIVES with probe `kill -0 8811`.
Provider compacts 40k tokens later. Post-compact SessionStart reinjects S08
first: the continued session's FIRST tool call is the probe - the log has
grown, the job is alive, the wave resumes. Without the registry the resumed
session would have discovered the job by accident or re-run a 20-minute suite
blind. This is failure mode M1 handled as designed.

### N3. The crash-during-flush recovery

Host loses power between F3 (file written through S08) and F4 (no FLUSH-CLOSED
line). Next boot's SessionStart finds `handoff-20261114-031200-site-a.md` with
S01-S08 present, S09-S11 missing. Reinjection runs on the prefix (marked
`PREFIX-ONLY sha=none`), and the resumed session completes S09 (next actions
were still derivable from the tasklist file), S10, S11, appends ADDENDUM,
closes with the hash. No item lost: the incremental append (F3) guaranteed the
prefix was already on disk before the crash.

---

## PART O - PROBE-SCRIPT OUTPUT CONTRACTS (normative)

probe_items.py:
- exit 0 + `PLANTED 12/12 present; FTS5 hits >= threshold` when green passes.
- exit 1 + `MISSING: <comma-separated item ids>` when below threshold.
- `--no-secret-values` flag: additionally asserts zero value-shaped strings
  (the G3 regex set) anywhere in the reinjection or handoff output; a value hit
  exits 2 (distinct from a miss - a leak is a different failure than a loss).

check_handoff.py:
- validates, in order: file exists; contains FLUSH-CLOSED with a sha256 that
  matches the file body recomputed at check time; contains exactly 11 section
  headers S01..S11 in order; the checklist section (S11) reports 11/11 PRESENT.
- exit 0 only if ALL hold; each failed check prints its own numbered line so
  the failure report is diffable between runs.

sim_fill.py:
- `--leg red|green`, `--window N`, `--fill 0.90`, `--trip-at N`
  (auto = WINDOW - RESERVE), `--role writer|receiver` for the cross-host drill.
- prints the ledger ramp lines with the fill value at every step, so the trip
  band is auditable from the transcript alone.
- exit codes: 0 green pass; 3 red-leg expected failure (survivors < 12 - this
  is the SUCCESS exit for the red leg, distinguishable from a crash); any other
  non-zero = the harness itself failed.

---

## PART P - OPERATIONAL RUNBOOK (post-install, for humans and agents)

P1. Manual force-flush (owner or agent request, e.g. before /clear): invoke the
    skill with `force: true`; the trigger math is skipped, the flush protocol
    runs in full, verification still gates. Manual flushes produce the same
    artifact class as automatic ones - one format, one store, one audit path.

P2. After ANY provider compaction observed mid-session: the first action is the
    post-compact reinjection leg (E1). If the harness exposes no hook, the agent
    runs the reinjection query manually as its first post-compact action.
    Symptom of skipping this: the session re-asks for credentials or re-explores
    known paths within the first 5 turns post-compaction. That symptom is a
    runbook hit, not a novel bug: check whether the newest handoff exists and
    was reinjected (the dedup guard may have suppressed it - check
    last_reinjected_sha).

P3. Weekly: run the store hygiene pass - orphan handoff files (in HANDOFF_DIR,
    absent from handoff_meta by sha256) are re-indexed per H-lag; handoff files
    older than 30 days are cold-archived (moved, never deleted, pointer left in
    the store body).

P4. If a session EVER says "I don't have X" for something predating its last
    compaction: that is a P1 defect against this skill, not a user error. Log
    it (symptom -> cause -> fix) in the runbook, reproduce with sim_fill.py,
    fix, re-run the green leg. The skill's whole reason to exist is the absence
    of that sentence.

P5. Escalation: two consecutive loss-diff failures in a real (non-sim) session
    -> hard abort per Part F; surface to the operator with the diff output
    attached; do not let the session continue under compaction pressure with a
    broken gate.

---

## PART Q - FAQ (locked at spec time)

Q: Why not just rely on the harness's session export files?
A: Exports are byte dumps, not curated state - they lack completeness rules,
   the secret gate, FTS5 retrieval, and the pre/post diff. An export is
   evidence; the handoff is the verified extract.

Q: Why 11 sections and not fewer?
A: Each section maps to one measured loss class. Fewer sections re-opens a
   measured failure class; more adds ceremony without a failure to point at.
   Section boundaries are frozen with the format spec (I3).

Q: Why is the red leg mandatory before green on every host?
A: A gate that was never seen failing on its own defect is prose, not a guard
   (measured fleet rule: re-introduce the defect and watch the gate exit
   non-zero). The red leg is that re-introduction, per host, because sqlite
   builds and sync layers differ per host.

Q: Does this replace CONTINUE-HERE files?
A: No. CONTINUE-HERE is the owner-facing continuity artifact written at task
   end; the handoff is the machine-facing loss-protection artifact written at
   the fill line. They share section content but differ in audience and
   trigger. The handoff's Section 6 points at CONTINUE-HERE files so the two
   never diverge silently.

Q: What if the provider window shrinks mid-run (model switch)?
A: Arm-time values are pinned in compaction-state.json; a window change
   requires a re-arm, and the re-arm logs the old vs new trigger lines into the
   session log and the next handoff's S01. Fill already above the NEW trigger
   fires the flush on the next gauge tick.

---

## PART R - TRACEABILITY MATRIX (owner requirement -> skill part)

Owner requirement, verbatim: "track the context window; when fill hits ~850-900k
on a 1M window, AUTOMATICALLY compact BUT save EVERYTHING first - decisions,
facts, API-key REFERENCES never values, open threads, file paths, in-flight task
state - into the FTS5-indexed memory store, so the continued session never says
I don't have X, provide API keys again."

| # | Requirement clause                              | Satisfied in    |
|---|--------------------------------------------------|-----------------|
| 1 | Tracks the context window (e.g. 1M)             | Part B, C1-C2   |
| 2 | Automatic at ~850-900k on a 1M window           | C2-C5           |
| 3 | AUTOMATIC (no human/model decision needed)      | C4, alwaysFire: true |
| 4 | Saves EVERYTHING before compacting              | Part E F1-F8    |
| 5 | ...decisions                                    | S03             |
| 6 | ...facts                                        | S04 (src: pointers) |
| 7 | ...API-key REFERENCES, never values             | S05 + G3        |
| 8 | ...open threads                                 | S07 ([OWNER-BLOCKED] flags) |
| 9 | ...file paths                                   | S06 (absolute paths) |
| 10| ...in-flight task state                         | In-flight registry + S08 |
| 11| into the FTS5-indexed memory store              | Part H          |
| 12| the continued session never says "I don't have X"| Part E reinjection + Part F diff + P4 rule |
| 13| "provide API keys again" eliminated specifically| S05 pointer-first + E2(4) |
| 14| works on PC + both VPSes                        | Part K deployment table |
| 15| lossless (nothing skipped)                      | Part F pre/post diff, 11 sections, zero MISSING gate |

Any future edit that breaks a row of this matrix without updating the matrix
(and the frozen-spec hash) is itself a spec violation.

---

## PART S - INTEGRATION SMOKE-MATRIX (one line per pair)

| Pair                                        | Shared surface                    | Smoke check                                            |
|---------------------------------------------|-----------------------------------|--------------------------------------------------------|
| auto-lossless-compaction <-> context-budget-manager | fill_tokens, window, reserve | same reserve value read by both; trip band matches formula |
| auto-lossless-compaction <-> session-memory-auto    | memory.db, FTS5 schema          | handoff rows visible to session-memory-auto's query path |
| auto-lossless-compaction <-> context-handoff        | 11 section names + order        | check_handoff.py enforces the exact S01..S11 sequence  |
| auto-lossless-compaction <-> lesson-recall          | Section 10 lesson format        | a post-flush spawn brief contains S10 lessons verbatim |

---

## PART T - FAQ (locked at spec time)

See the FAQ in the source spec; the four locked answers (exports vs handoffs,
why 11 sections, why red-before-green per host, CONTINUE-HERE coexistence) and
the model-switch re-arm rule are reproduced in PART Q above and are frozen.

---

## PART U - REFERENCES (fleet-internal)

- context-budget-manager skill: per-turn fill accounting, cumulative ledger.
- session-memory-auto skill: capture leg, observation store, FTS5 corpus.
- context-handoff skill: the 11-section handoff format (format owner).
- lesson-recall skill: mechanical lesson injection at spawn briefs.
- autocompact-750k-fleet (memory): window clamping + provider trigger math.
- sync-prune skill: catalog/index deletion propagation rules (registration path).
- secret-rotation skill: response protocol if G3 ever lets a value through.
- skill-frozen-spec skill: hash-locking the source spec at install (I5).

## PART V - REJECTION CRITERIA (when NOT to trust a "flush done")

- A handoff file without a FLUSH-CLOSED line is a PREFIX, never a completed
  flush; do not treat it as closed state (M2 governs completion).
- A checklist that says 11/11 without the S11 evidence (row counts, sha256,
  duration) is a self-grade, not evidence; reject it.
- An FTS5 row count that did not grow by >= 12 rows for this flush means the
  index write silently failed; the flush is NOT done (M8).
- A reinjection that omits S05 secret pointers will cause the continued session
  to re-ask for credentials; that symptom is a P1 defect against this skill
  (runbook P4), not a user error.
- Any observed trip ABOVE 900,000 on a 1M window violates the owner band (C3)
  and fails the install test.

## PART W - INSTALL CHECKLIST (what the executor verifies on every host)

1. File present at `<skills>/auto-lossless-compaction/SKILL.md`, ASCII-only
   (no codepoint > 127), frontmatter intact, `alwaysFire: true` present.
2. Line count of the installed file >= 1,000.
3. FTS5 probe passes on the host's store: `SELECT fts5('test')` returns a
   row (or the explicit LIKE-fallback warning is configured, never silent).
4. The DDL from PART H applied to the host's memory.db (idempotent; safe to
   re-run).
5. Skill registered in the host's catalog index so the router can fire it
   (name, description, alwaysFire).
6. Red leg run BEFORE green leg on the host: naive path proven lossy
   (non-zero exit, MISSING list pasted), then green leg proven lossless
   (12/12 planted items, >= 11/12 FTS5 recall, checklist 11/11 PRESENT,
   trip inside the 850k-900k band on a 1M window).

## PART X - THE 12-ITEM PLANTED-STATE STANDARD (test contract)

The green proof plants exactly 12 known items in a simulated session:
- 3 decisions (each with why + a rejected alternative) -> land in S03.
- 3 sourced facts (each with a src: pointer) -> land in S04.
- 2 secret pointers (name -> pointer, zero values) -> land in S05.
- 2 file paths (absolute) -> land in S06.
- 1 open thread (status + unblock condition) -> land in S07.
- 1 in-flight job (probe command + keep-or-kill verdict) -> lands in S08.
- next actions (ordered, each with action + check) -> land in S09.
The post checklist must account for all 12; the FTS5 keyword query must
retrieve >= 11/12 (>= 92% recall); any allowed miss must be asserted and
explained (stopword-collision artifact class). The red leg (naive compaction:
keep last 50 turns, drop the rest, no handoff, no index) must demonstrably
lose planted items - a guard never seen failing is prose, not a guard.

---

## PART Y - SECTION WRITING TEMPLATES (the exact skeleton an agent fills during F3)

These templates are the mechanical form of the completeness rules in F3. An
agent writing the handoff fills each template line-for-line; a line that
cannot be filled honestly becomes the verified-NONE line for that field.
Nothing outside these templates goes into the handoff file.

### Y1. TEMPLATE S01 - SESSION IDENTITY AND WINDOW MATH

```
## S01 - SESSION IDENTITY AND WINDOW MATH
session_id: <id>
host: <pc | vps-webdev-179.61.138.8 | vps-knowledge-179.61.251.98>
model: <provider/model id>
started_at: <ISO-8601 UTC>
window: <tokens>
reserve: <tokens>
trigger_line: <tokens>
fill_at_flush: <tokens>
compact_due: <reason, e.g. "fill 852400 >= trigger 850000">
```

Completeness check before writing the next section: 9/9 fields non-empty.

### Y2. TEMPLATE S02 - TASK AND GOAL STATE

```
## S02 - TASK AND GOAL STATE
goal_verbatim: |
  <the exact goal text as declared, never paraphrased>
wave_phase: <current wave or phase>
stop_condition: <the compound stop condition, verbatim>
progress: <done>/<total> done, <n> in-flight, <n> remaining
```

Completeness check: goal_verbatim non-empty AND progress line present.

### Y3. TEMPLATE S03 - DECISIONS MADE AND WHY

```
## S03 - DECISIONS MADE AND WHY
D-<n>: decision=<what was chosen>
  rejected=<alternative that was rejected>
  why=<the reason, one line>
  evidence=<tool result / file / URL pointer>
```

Completeness check: >= 1 decision block with a why AND a rejected
alternative, or the literal `NONE (verified: nothing to record)`.

### Y4. TEMPLATE S04 - FACTS LEARNED (BY REFERENCE)

```
## S04 - FACTS LEARNED (BY REFERENCE)
F-<n>: fact=<one-line statement>
  src: <URL | absolute file path | command + output hash | doc section>
  verified: <date>
```

Completeness check: every F-line contains `src:`. A fact with no source is
written with `src: UNVERIFIED - no pointer` and is still a loss-class defect
worth fixing during the flush.

### Y5. TEMPLATE S05 - SECRETS AS POINTERS (NEVER VALUES)

```
## S05 - SECRETS AS POINTERS (NEVER VALUES)
SEC-<n>: <name> -> <pointer (env var | vault path | config file + JSON path |
  keychain entry | service + owner contact)>
```

Completeness check: every line matches `name -> pointer`; the G3 value-shape
scan over this section returns zero hits. A hit FAILS the flush (G3).

### Y5. TEMPLATE S06 - FILE PATHS AND ARTIFACTS

```
## S06 - FILE PATHS AND ARTIFACTS
P-<n>: <absolute path> | <one-line description> | <created|modified|deleted>
  | <verification: built? tested? untested?>
```

Completeness check: >= 1 line or verified-NONE; every path absolute (no
relative paths, no `~` - a fresh process on another host must resolve them).

### Y6. TEMPLATE S07 - OPEN THREADS

```
## S07 - OPEN THREADS
T-<n>: <thread> | why_open: <reason> | unblock: <exact condition>
  | owner: <me | owner | external-service | another-session>
  | flag: [<OWNER-BLOCKED: <the single named item>>]
```

Completeness check: every thread has status + unblock condition.

### Y7. TEMPLATE S08 - IN-FLIGHT COMMANDS AND PROCESSES

```
## S08 - IN-FLIGHT COMMANDS AND PROCESSES
J-<n>: pid=<pid> host=<host>
  command: <command line>
  log: <absolute log path>
  started_at: <ISO-8601 UTC>
  expected_side_effect: <what growing output proves it is alive>
  probe: <the EXACT probe command>
  verdict: <SURVIVES | MUST DIE> (if MUST DIE: kill_command=<command>)
```

Completeness check: every entry has a probe command and a verdict.

### Y8. TEMPLATE S09 - NEXT ACTIONS (ORDERED)

```
## S09 - NEXT ACTIONS (ORDERED)
1. action: <command or edit target> | check: <pass/fail check>
2. action: <command or edit target> | check: <pass/fail check>
...
```

Completeness check: ordered list; every step has an action AND a check.

### Y9. TEMPLATE S10 - FAILURES TRIED AND RULES LEARNED

```
## S10 - FAILURES TRIED AND RULES LEARNED
L-<n>: symptom: <what was observed>
  cause: <root cause>
  fix: <what was done / what to do next time>
  gate_fired: <gate name + why, if any>
```

Completeness check: >= 1 lesson or verified-NONE.

### Y10. TEMPLATE S11 - VERIFICATION EVIDENCE AND HANDOFF CHECKLIST

```
## S11 - VERIFICATION EVIDENCE AND HANDOFF CHECKLIST
checklist:
  S01 PRESENT | S02 PRESENT | S03 PRESENT | S04 PRESENT | S05 PRESENT
  S06 PRESENT | S07 PRESENT | S08 PRESENT | S09 PRESENT | S10 PRESENT
  S11 PRESENT
fts5_rows_delta: <n>
sha256_at_close: <hash>
flush_duration_seconds: <n>
diff_result: PASS (per-class counts: <class>=<pre>-><post> ...)
```

Completeness check: 11/11 PRESENT; sha256 recorded; diff PASS with counts.

### Y11. The FLUSH-CLOSED line (F4, appended after S11)

```
FLUSH-CLOSED sha256=<hash> sections=11/11 fill_before=<F>
```

---

## PART Z - CROSS-HOST DRILL (the receiver-side procedure, normative)

Purpose: prove the continued-session guarantee is host-INDEPENDENT - a handoff
written on one host is recovered on another, and zero secret VALUES cross.

Z1. WRITER side (e.g. VPS web-dev): run the green leg; the handoff file lands
    in HANDOFF_DIR and the FTS5 rows land in the LOCAL store. Announce
    `cross-host writer done handoff=<file>`.

Z2. Sync: the fleet git-sync layer propagates the handoff FILE to the other
    hosts (the DBs do NOT sync). If the file does not arrive, fix the git-sync
    layer, not the skill.

Z3. RECEIVER side (e.g. VPS knowledge): at SessionStart, run the lag repair
    (H-lag): compare handoff_meta MAX(created_at) against the newest synced
    file under HANDOFF_DIR; re-index unindexed files by sha256 membership
    (not timestamps - clock skew rule, G7).

Z4. Run the reinjection query (H-reinject) on the receiver against the synced
    handoff: assert sections 2, 5, 7, 9, 10 are present in the reinjected
    block.

Z5. Secret scan: run the value-shape regexes (G3) over the receiver's
    reinjection output; assert 0 value-shaped strings. A value crossing hosts
    is a STOP-everything event: rotate the secret (secret-rotation skill),
    fix G3, re-run.

Z6. Dedup proof: a second SessionStart on the receiver's lineage must NOT
    double-inject (last_reinjected_sha matches; only delta hits injected).

Z7. Record the verdict line:
    `cross-host drill: writer=<host> receiver=<host> sections=2,5,7,9,10
    secret_scan=0_values reinjection=OK`

A same-host-only proof is level PARTIAL: the owner's requirement is that
"the continued session never says I don't have X" holds on WHATEVER host
resumes, so the receiver must be a different host than the writer.

---

## PART AA - FLUSH TRANSCRIPT TEMPLATE (what a compliant flush prints)

A compliant flush leaves these exact line shapes in the session log, in this
order (auditors and the install test grep for them):

```
auto-lossless-compaction SOFT fill=<F>/<W> (>=70%). Ledger must stay fresh; finish or checkpoint long-running work now.
auto-lossless-compaction FLUSH START fill=<F> trigger=<T>
auto-lossless-compaction FLUSH DONE handoff=<file> sha256=<hash> fts5_rows=<N> checklist=11/11 PRESENT. Compaction permitted.
```

and, when the provider does not auto-compact first:

```
auto-lossless-compaction FLUSH OK fill=<F> handoff=<file> store_rows=<N>
```

The SOFT warning (C4 step 3) fires ONCE per crossing, not per turn; the trip
line is never repeated after the flush completes unless fill climbs above the
trigger again after fresh post-flush work (M6).

---

## PART AB - ARM-TIME SELF-TEST (runs once per host at install, then weekly)

The arm block runs this 4-line self-test at install and logs the result:
1. Formula check: for each window in the C2 table, compute RESERVE and
   TRIGGER from the formula and assert they equal the table row. Any
   mismatch = refuse to arm.
2. Store check: create the DDL (idempotent) and run
   `SELECT COUNT(*) FROM memory_fts`; it must return an integer without
   error (FTS5 probe).
3. Registry check: compaction-state.json is creatable and parseable in
   HANDOFF_DIR.
4. Handoff dir check: HANDOFF_DIR exists or can be created; the last successful
   handoff (if any) is discoverable by sha256 in handoff_meta.
Any failed line = refuse to arm + loud warning; a broken guard breeds false
confidence, which is worse than no guard.
