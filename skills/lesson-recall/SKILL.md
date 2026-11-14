---
name: lesson-recall
description: Mechanical lesson-recall injection at spawn briefs - the confabulation guard. Before ANY subagent spawn, a deterministic recall step queries the lesson store (the memory corpus: handoffs, brain notes, observations index, runlogs) for lessons matching the mandate's domain and injects the top-N verbatim into the spawn brief. Closes the confabulation class where a fresh-context seat confidently re-derives (or contradicts) a lesson the fleet already paid for - without relying on the spawner remembering to include prior art. Pairs with session-memory-auto (which builds the corpus) and subagents (which writes the briefs). alwaysFire - fires on every spawn brief assembly, before the brief leaves the orchestrator.
alwaysFire: true
---

# lesson-recall - inject the lessons the fleet already paid for

You are running **lesson-recall**. Every subagent brief gets a PRIOR ART
block, assembled by a deterministic query - not by the spawner's memory.
The block carries the lessons that match the mandate's domain: past
failures, established rules, prior decisions, and known gotchas, quoted
verbatim with their provenance. The child starts knowing what the fleet
knows.

**Why this exists (the measured failure class):** fresh context is the
point of subagents (de-correlation) - and its cost is amnesia. A fresh
seat re-derives the excerpt-truncation bug, re-runs the same search its
sibling ran, or violates the "never paste session history into briefs"
rule because no one told it - because the spawner forgot, because the
lesson lived in a note nobody opened. The confabulation variant is worse:
a seat confidently states a "fact" the fleet disproved weeks ago.
Mechanical recall at brief-assembly time closes the gap without relying on
anyone's discipline (the hooks-over-habits doctrine, applied to spawn
briefs).

**What this skill is NOT:**

- NOT a memory dump. The block is top-N (3-5) RELEVANT lessons, verbatim,
  capped in lines. A 2k-token blob of "context" buries the mandate - the
  measured dispatch-payload failure.
- NOT a replacement for the self-contained brief. The brief still carries
  everything the seat needs; the prior-art block is ADDITIVE - it prevents
  re-derivation and contradiction, it does not carry the task.
- NOT a judgment layer. Recall is mechanical (query -> rank -> inject);
  deciding HOW to use a lesson is the child's job. Conflicting lessons
  are both injected with their dates - the child sees the contradiction
  explicitly (Section 5).

---

## 0) HARD RULES (survive compaction)

1. **No spawn without the recall step.** The brief template has a
   `[prior-art]` slot; the assembly procedure fills it from the store. An
   empty result injects the marker line `[prior-art: none for <terms>]` -
   proving the step RAN, not that it found nothing.
2. **Verbatim quotes only.** Lessons are quoted with provenance
   (source file + date). A paraphrased lesson is a new lesson in the
   making - the exact confabulation this skill prevents.
3. **Top-N = 5, line cap = 40.** Overflow ranks by relevance and states
   the truncation (`+12 more; query: <terms>`), never silently drops.
4. **The spawner does not editorialize.** No "ignore lesson X" or
   "lesson Y is outdated" in the block - if a lesson is superseded, the
   store's supersedes chain shows it (session-memory-auto's current-value
   view); the block carries the CURRENT value, not the spawner's opinion.
5. **Provenance is mandatory.** Every lesson line carries
   `[<source>:<date>]` so the child (and the next audit) can verify the
   lesson against its source. An unprovenanced lesson is a rumor.
6. **Conflicts are injected, not resolved** (by the spawner). The child
   sees both sides + dates and must state which it follows and why in its
   final JSON - that statement is auditable evidence of judgment, not
   inherited bias.
7. **The block is evidence, not instructions.** Lessons inform; the
   MANDATE instructs. A lesson must never override the mandate (if it
   would, that is a spawner error - fix the mandate or supersede the
   lesson, before spawn).
8. **Skipped-recall spawns are counted.** The gates line reports
   `lesson-recall: skipped: <why>`; the audit counts skips the same way
   every gate counts bypasses. "I forgot" is a bypass, not a reason.

---

## 1) WHEN IT FIRES

Always-fire at BRIEF ASSEMBLY (subagents Wave-1 and every later wave):

| Spawn type | Recall query shape |
|-----------|--------------------|
| recon seat | domain terms + tool names of the recon surface |
| build seat | stack terms + the specific subsystem + known-gotcha terms |
| verify/adversarial seat | the artifact class + past review findings for that class |
| repair seat (recovery-engineers) | the failure signature + prior repair attempts for that seat class |
| pentest seat | target class + technique names + the engagement's known constraints |
| web-dev seat | the niche + the design-system rules + past deploy gotchas |

The query is built from the mandate text (top terms), mechanically - the
same terms the session-memory recall would use, so the corpus serves both
the parent's recall and the child's injection.

---

## 2) THE LESSON STORE - what counts as a lesson

The store is NOT a new database - it is the corpus this fleet already
maintains, queried through one lens:

| Source | What it yields | Example line |
|--------|---------------|--------------|
| brain knowledge notes | durable rules, decisions | `[web-dev/knowledge:2026-11-09] never ban text from client logo prompts` |
| docs/handoffs | task-scoped state, recent decisions | `[handoffs/2026-11-11:2026-11-11] excerpt tail anchoring: parse full final text` |
| observations store (DECISION/GOTCHA/FAILURE types) | session-level lessons | `[obs#388:2026-11-09] pkill self-kill: bracket-trick the pattern` |
| runlog post-mortems | campaign-scale lessons | `[runlog-DR9:2026-11-12] LLM hypotheses need deterministic confirmation` |
| per-skill known-failure tables | operational recipes | `[scope-enforcer:2026-11-13] fail-closed on scope-load-fail` |

Selection criteria for a line to be INJECTED (vs. available):

- It matches >= 1 query term (mechanical match).
- It is CURRENT (not superseded; the supersedes chain from
  session-memory-auto's observations, or date-dominant for corpus files).
- It is ACTIONABLE for this mandate (a rule, a gotcha, a decision - not
  background prose).

---

## 3) THE PROCEDURE (step by step)

**S1. Extract the query terms.** From the mandate text: top 5-8
content terms (drop stop-words; keep tool names, stack names, domain
nouns). Deterministic: same mandate -> same terms.

**S2. Query the store.** The FTS query (session-memory-auto's smem
recall) over the corpus + observations, filtered to lesson-type rows:

```
smem recall "<terms>" --kind lesson --top 8
```

(corpus rows qualify as lessons when their section matches
lesson-headers: RULE / GOTCHA / DECISION / FAILURE / GOTCHAS; the
filter is a header grep, not a judgment.)

**S3. Rank and cap.** Order: cwd/mandate-term matches, recency within
class, DECISION > GOTCHA > RULE for actionable-ness (the ordering is
fixed in the script, versioned). Cap at 5 lines / 40 lines of block.

**S4. Compose the block.** Fixed shape:

```
[prior-art] (3 of 5 matched; terms: excerpt, tail, parse)
1. [RULE handoffs/2026-11-11 2026-11-11] parse the FULL final text; when
   excerpting, anchor the tail at the evidence (gates line).
2. [GOTCHA obs#388 2026-11-09] pkill self-kill: bracket-trick patterns.
3. [DECISION runlog-RGAP1 2026-11-08] anchor excerpts at the max-field
   gates line within last ~1600 chars.
(+2 more; query: "excerpt tail parse")
```

**S5. Inject into the brief** at the fixed slot (after MANDATE, before
OUT). The brief stays self-contained; the block is additive evidence.

**S6. Child acknowledgment contract.** The child's final JSON gains one
field: `"lessons_followed": [<ids>]` (and, when a conflict existed,
`"conflict_resolution": "<which and why>"`). This makes lesson-following
auditable - a repair seat that re-commits a documented gotcha is caught
by its own JSON.

**S7. Record.** The gates line: `lesson-recall: fired (3 injected, 0 conflicts)`.

---

## 4) VERIFICATION COMMANDS

```
# V1. The recall step ran for every spawn in the wave
grep -c "prior-art" <briefs-dir>/*.md          # expect == spawn count

# V2. Provenance present on every line
grep -c "\[.*:20[0-9][0-9]-" <briefs-dir>/<brief>.md   # >= lesson count

# V3. Line cap honored
awk '/\[prior-art\]/,/^\(^[^0-9[]|\z/' <brief>.md | wc -l   # <= 42

# V4. The marker-on-empty rule (guard-proves-itself)
#    once per rollout: craft a nonsense-terms mandate, spawn a probe,
#    confirm the brief contains "[prior-art: none for <terms>]" - the
#    step RAN. If the marker is missing, the recall step is not wired.

# V5. No editorializing
grep -cE "ignore lesson|outdated lesson|skip prior" <brief>.md  # expect 0
```

V4 is the once-per-rollout self-attack (the same discipline as review V3
and scope T10): a recall step that has never demonstrated the
empty-result marker may not actually be wired - "no lessons found" and
"step never ran" look identical without the marker.

---

## 5) CONFLICT HANDLING (the interesting case)

When the store holds contradictory lessons (the fleet changed its mind,
or two domains disagree), the block carries BOTH:

```
3. [DECISION runlog-DR2 2026-11-02] prefer deterministic sqlmap over
   LLM-crafted payloads for blind injection.
4. [RULE brain/cybersec 2026-10-14] document the payload logic before
   running external tools.  (SUPERSEDES note: see DR2 2026-11-02 for the
   payload-authoring exception)
```

The child must resolve in its JSON (`conflict_resolution`) - and that
resolution is reviewable. The spawner resolving the conflict silently
would (a) bias the child, (b) hide the fact that the fleet's doctrine
changed, (c) produce a child whose reasoning cannot be audited. The
measured failure (children confidently citing outdated rules) came
exactly from pre-resolved injection.

Superseded lessons are INJECTED ONLY with their supersede note - a bare
superseded lesson is store corruption (session-memory-auto's
current-value view is the source of truth for the observations; corpus
files get the note by date-dominance).

---

## 6) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Brief has no prior-art block | assembly skipped the step | V1; the template slot is mandatory; marker-on-empty proves the step |
| Block is 60 lines of prose | cap not enforced / background prose admitted | top-5 + line cap; filter to lesson-headers only |
| Child cites a disproven rule | superseded lesson injected bare | supersedes chain must be resolved before injection (current-value view) |
| Child contradicts an injected lesson silently | acknowledgment contract missing | `lessons_followed` in final JSON; review checks the field |
| Recall returns garbage matches | query terms too generic | mandate terms are content terms; if the mandate itself is vague, that is a clarify defect (spec-clarify-gate) |
| Same lessons injected to every seat regardless of mandate | cached query, not per-mandate | query derives from THE mandate text; V-check: two different mandates -> different blocks |
| Block cites lessons from an unrelated box's context | corpus crawl pulled foreign cwd | cwd weighting in recall; corpus sources carry their box in provenance |

---

## 7) ANTI-PATTERNS

- THE CONTEXT BLOB: injecting "relevant background" paragraphs. Lessons
  are lines with provenance; a blob is the dispatch-payload failure.
- THE SPAWNER'S TAKE: "lesson 2 is outdated, skip it." Then supersede it
  in the store (one write) and let the chain show it - never silent
  curation at dispatch time.
- THE UNVERIFIED LESSON: injected lines without provenance. A lesson is
  a claim about the fleet's experience; claims are sourced or they are
  hunches, and hunches get confidence labels, not authority.
- THE CHILD THAT OBEYS LESSONS OVER THE MANDATE: lessons are evidence;
  when they conflict with the mandate, the child states the conflict in
  its JSON - it does not silently pick either.
- THE FORGOTTEN RECALL: spawning "quickly" without the step. The marker
  system exists so the skip is visible in the brief itself.

---

## 7) WORKED EXAMPLES

### 7.1 The recon seat that almost repeated the truncation bug

Mandate: "harvest session data via cursor-paginated REST; report gate-line
compliance per session." Query terms: cursor, paginated, rest, gate,
parse. Recall returns (3):

1. [RULE handoffs 2026-11-11] parse the FULL final text; excerpt tails
   produced false 0/2 STRICT counts.
2. [GOTCHA obs#411 2026-11-10] deadline-sampled excerpts corrupt rows;
   harvest only after terminal outcome.
3. [DECISION runlog-RGAP1 2026-11-08] dedup fires by tool-call id.

The seat's final JSON: `lessons_followed: [1,2,3]`, full-text parsing
confirmed in its method section. Without the block, the seat's natural
implementation (4000-char excerpts - the obvious choice) reproduces the
measured false-FAIL class, and the review would find it only after the
corrupt rows existed.

### 7.2 The repair seat with prior attempts injected

recovery-engineers dispatches a fresh repair seat (3rd+ fix, fresh
worker). The brief's prior-art block carries the prior attempts summary
(the mandated prior-attempts file) AS A LESSON:

1. [FAILURE repair-log 2026-11-13] attempt 1: same spawn, same failure -
   retry is not repair (banned corrective prompt class).
2. [GOTCHA obs#502 2026-11-13] the seat's failure was unresolved ${VAR}
   (Gate A) - preflight was skipped on that spawn.

The repair seat now designs around the actual root cause instead of
repeating the respawn. The recovery-engineers skill's core promise -
"designed, not repeated" - is mechanically supported at brief time.

### 7.3 Conflicting lessons, resolved by the child

A web-dev seat's mandate: rebuild a component for performance. Store:
[RULE brain/web-dev 2026-10-20] "prefer CSS transforms for animation" and
[GOTCHA obs#455 2026-11-05] "this component's animation is JS-driven for
a11y reasons (reduced-motion handling)". The child states:
`conflict_resolution: keeping JS path (obs#455) - a11y beats the general
rule; performance gain realized via transform-only properties in the JS
path`. The reviewer reads that line and adjudicates in seconds - the
conflict was visible, the resolution reasoned, the general rule honored
where it applies.

---

## 8) GATES-LINE CONTRACT

Slot in the per-task extension:

```
lesson-recall: fired (3 injected, 0 conflicts)
lesson-recall: fired (4 injected, 1 conflict resolved by child)
lesson-recall: marker-on-empty (0 for <terms>)
lesson-recall: skipped: no spawn this turn
lesson-recall: skipped: store unwritable/unavailable (grep fallback used)
```

Feeds `dim-correctness` (child JSON shows followed lessons - re-derived
bugs caught) and `dim-completeness` (prior-art coverage of the mandate
domain). The grep-fallback (store unavailable) is honest degradation:
query docs/handoffs + brain notes by grep, mark the block
`[prior-art: grep-fallback]`, say so.

---

## 9) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| session-memory-auto | builds and indexes the store; its recall query is the engine; its DECISION/GOTCHA/FAILURE types are the lesson classes |
| subagents | brief assembly calls this skill before dispatch; the brief template's prior-art slot is mandatory |
| recovery-engineers | repair briefs carry the failure signature + prior attempts as lessons (the anti-respawn evidence) |
| auto-review-gate | the reviewer checks: did the child's method honor the injected lessons? A documented gotcha re-committed is a Major |
| spec-clarify-gate | SPEC.md R-lines are query terms too - spec-level lessons (requirements decisions) inject into briefs |
| honestscore | "NOT verified" items get checked against the store: a contradicting lesson makes the item verifiable |
| prompt-preflight | validates the assembled dispatch payload - the prior-art block must survive preflight's payload checks (no bloat, no broken template) |
| ideas | Step 0 (mistake-memory preflight) is the parent-side version of this skill; lesson-recall is the child-side delivery |

---

## 10) OPEN ITEMS (honest limits)

- No head-to-head benchmark exists for brief-injection vs plain RAG
  memory (the DR7 caveat inherited from session-memory-auto); the design
  bet is that VERBATIM + provenance + top-5 beats both free-form context
  and nothing. The veto/conflict-resolution JSON field is the quality
  signal to watch.
- Relevance ranking is heuristic (term match + recency + type); the
  confabulation cases it misses will look like "child invented X" with no
  lesson retrieved - tune the query-term extraction when that happens.
- The store's supersedes chains are only as good as the writers'
  discipline (session-memory-auto's consolidation pass D09 would automate
  this); until then, date-dominance is the fallback and it is noted on
  every injected corpus lesson.

---

## APPENDIX A - THE ONE-CARD OPERATING SUMMARY

```
EXTRACT  mandate -> 5-8 content terms (deterministic, same text = same terms)
QUERY    smem recall "<terms>" --kind lesson --top 8
RANK     term match + recency + type (DECISION > GOTCHA > RULE), versioned
COMPOSE  verbatim lines + provenance; cap 5 lessons / 40 lines; state
         truncation; inject supersedes notes on old corpus lessons
INJECT   [prior-art] slot in the brief (after MANDATE, before OUT);
         empty result -> marker line proving the step RAN
CONTRACT child final JSON: lessons_followed[] + conflict_resolution
         (when conflicts existed) - auditable, not inherited bias
VERIFY   V1 block per spawn | V2 provenance | V3 line cap |
         V4 marker-on-empty (once per rollout) | V5 no editorializing
NEVER    context blobs | silent curation | mandate overrides by lesson |
         provenance-free lines | forgotten recall
```

---

## APPENDIX B - BRIEF TEMPLATE WITH THE SLOT (subagents composition)

```
## MANDATE
<one sentence, IN-scope>

[prior-art block - filled by lesson-recall, fixed shape]

## OUT (explicit exclusions + sibling scopes with do-not-cover)

## DELIVERABLE
<named artifact + format + acceptance test>

## PINNED ASSUMPTIONS
<names, interfaces, conventions decided BEFORE dispatch>

## EVIDENCE RULE
paste real tool results; no run -> [UNVERIFIED]

## CONTEXT HYGIENE
no secrets, carry only what the scope needs

## REPORT
blocker in one line; no scope widening; no subagent spawning;
final JSON: {"status","deliverable_path","findings","blockers",
"files_touched","lessons_followed","conflict_resolution"}
```

The slot's position (after MANDATE, before OUT) is deliberate: the child
reads the lessons AS PART OF understanding the task, not as an appendix
to skip. The JSON field is added by this skill's contract and consumed
by review.

---

## APPENDIX C - QUERY-TERM EXTRACTION (the deterministic recipe)

```
1. tokenize the mandate (lowercase, strip punctuation)
2. drop stop-words (the fixed list in the script header)
3. keep: tool names, stack nouns, protocol names, domain terms
4. cap at 8, preserving mandate order (first-mentioned = highest weight)
5. add the SPEC R-keywords when a SPEC.md exists (R-lines' key phrases)
```

Same mandate -> same terms -> same block: determinism is what makes the
recall auditable (a re-run produces the same injection, so a reviewer can
verify what the child actually saw). A non-deterministic recall is a
different experiment every spawn - useless for auditing.

Class hints (mechanical, from the spawn type): recon adds the surface's
tool names; build adds stack terms from the repo (package.json,
pyproject, csproj); pentest adds the engagement's target class; web-dev
adds the niche + design-system names.

---

## APPENDIX D - WORKED EXAMPLE: THE REPAIR BRIEF (full assembly)

Failure: build seat 3 failed twice on "the exporter loses rows over
100MB". recovery-engineers assembles a fresh repair seat.

```
S1 terms: exporter, rows, large, streaming (from failure sig + mandate)
S2 recall:
   1. [GOTCHA obs#388 2026-11-09] excerpt/tail truncation in large
      outputs - stream, never buffer (class: large-file handling)
   2. [FAILURE repair-log 2026-11-13] attempt 1 same spawn -> same
      failure; retry banned
   3. [DECISION spec R3 2026-11-13] streaming above 100MB is a spec
      requirement, not an optimization
S3 compose: 3 lessons, 9 lines, terms quoted in header
S4 inject + dispatch
S5 child JSON: lessons_followed [1,2,3], method states streaming +
   chunk verification, conflict_resolution: null
```

The repair converges in one dispatch. Without the block: the fresh seat
buffers (the obvious implementation), fails the same way, and the
15-loop abort clock starts ticking - the exact expensive class the
recovery skill measures.

---

## APPENDIX E - LESSON LIFECYCLE (write, supersede, retire)

```
WRITE   a session hits a gotcha / makes a decision / fails measurably
        -> observation (session-memory-auto capture) or brain note
        -> becomes queryable within one corpus-index pass
SUPERSEDE  a newer lesson contradicts an older one -> the new one sets
        supersedes=<old id> (observations) or the brain note updates the
        rule in place with a date (corpus)
RETIRE  consolidation pass (D09) prunes stale/duplicated lessons;
        retired lessons stay in the store with a retired note (never
        silently deleted - the history is evidence)
```

The lifecycle is shared with session-memory-auto by design: lesson-recall
consumes; session-memory-auto produces; the consolidation pass (D09) is
the missing automation, noted in both files. Until it exists, the recall
query's date-dominance + supersedes notes are the honest mechanism.

---

## APPENDIX F - WHAT MAKES A LESSON INJECTABLE (quality bar)

Not every matching row deserves the block. The bar:

- ACTIONABLE: states a rule/gotcha/decision with a DO or DON'T - not
  background ("the project uses Astro" is pinned context, not a lesson;
  the brief carries pinned assumptions separately).
- VERIFIED or MEASURED: a tool result backs it (hunch-tier lessons carry
  their confidence explicitly - the child weights them accordingly).
- RECENT-or-CURRENT: not superseded; corpus lessons within ~90 days
  unless doctrine (brain rules are timeless by intent).
- SCOPED: the lesson states its scope ("in Next.js 15", "for PrestaShop
  modules") - an unscoped lesson is a rule that will be over-applied.

The quality bar keeps the block small and dense: 5 lessons that change
behavior, not 5 facts that fill space.

---

## APPENDIX F - CONFABULATION CASES (what this skill catches, named)

1. RE-DERIVATION: child re-implements the buggy approach the fleet fixed
   (truncation, buffering, sync-over-async) - caught by lessons_followed.
2. STALE-FACT CONFABULATION: child states "the API requires X" when the
   store holds a measured "X was removed 2026-10" - caught by the
   reviewer reading the injected lesson vs the child's claim.
3. SIBLING REPEAT: child re-runs the same search a sibling ran - caught
   by the cross-seat observation trail (session-memory-auto Appendix H).
4. DOCTRINE DRIFT: child follows a superseded rule because the spawner
   curating from memory pre-picked it - prevented by verbatim+provenance
   + supersedes-note injection.
5. THE MISSING PRIOR ART: child invents a requirement the spec already
   answers - spec R-keywords in the query terms close this class.

Each case has a counter in the contract (marker, verbatim, conflicts,
JSON field) - the skill's design is "every failure mode has a visible
trace", not "children are smart now".

---

## APPENDIX G - METRICS (one line per wave)

```
recall | <date> | wave <n> | briefs <count> | injected <total>
| conflicts <count> | marker-empty <count> | skipped <count>
| avg-lines <n> | grep-fallback <yes|no>
```

Signals: injected-per-brief (persistent 0 = the extraction is bad or the
store is empty - check which); conflict rate (high = doctrine churn, feed
consolidation; zero forever = check the store is actually being read);
skip rate (nonzero skips on spawns = bypass audit). Same calibration file
as the other gates.

---

## APPENDIX G - FAQ (short, factual)

**Q: The mandate is unique - no prior art exists.**
The marker line (`[prior-art: none for <terms>]`) proves the step ran.
That is the success state for a novel task - the fleet will WRITE the
lessons during it.

**Q: The child ignores the lessons?**
Its JSON must list what it followed; a reviewer cross-checks. A child
that neither follows nor contradicts is a brief-quality finding (the
lessons were irrelevant - tune the query).

**Q: The spawner disagrees with an injected lesson?**
Rule 0.4: no editorializing. If the lesson is genuinely outdated,
supersede it in the store BEFORE spawn (one INSERT with supersedes); the
next recall carries the current value. Never editorialize per-brief.

**Q: Cost of the block on every spawn?**
~120-200 tokens per brief (5 lessons, capped). The measured alternative -
re-derivation, failed spawns, conflicting children - costs orders more.
The cap is what keeps it additive rather than burdensome.

**Q: Solo builds (no spawns)?**
Skill skips (`skipped: no spawn this turn`). The parent-side version of
the same discipline is ideas Step 0 + the session-memory recall before
designing - the lesson-recall contract starts at spawns, not at solo
turns.

---

## APPENDIX H - FINAL SELF-CHECK (before dispatch)

1. Does every brief in the wave carry the prior-art block (or marker)?
2. Is every lesson line verbatim with provenance?
3. Is the block within the caps (5 lessons / 40 lines)?
4. Are conflicts dual-injected with supersede notes?
5. Does the child JSON contract include lessons_followed?

Any "no" = fix before dispatch. The block costs ~150 tokens; the
confabulation it prevents costs a re-derived failure - the exchange rate
is the reason the step is always-fire, not best-effort.

---

## APPENDIX I - THE CONFABULATION GUARD PROVEN (install test)

Once per rollout, prove the step exists (the guard-until-proven rule):

1. Craft a mandate matching a KNOWN stored lesson (e.g. excerpt
   truncation).
2. Assemble the brief via the recall step; confirm the lesson appears
   verbatim with provenance.
3. Craft a nonsense-terms mandate; confirm the marker-on-empty line.
4. Confirm the child JSON contract in the brief template.

If step 3 fails (no marker), the recall step is decoration - the brief
pipeline writes prior-art by memory, which is the confabulation class
wearing a costume.

---

## APPENDIX J - THE DESIGN BET, STATED HONESTLY

The fleet has TWO memory paths now: the corpus recall (parent-side,
session-memory-auto) and the brief injection (child-side, this skill).
The unmeasured risk is redundancy - both firing on the same terms. The
design resolution: the parent recalls to DECIDE, the child receives to
EXECUTE - same store, different contracts. The veto/conflict JSON gives
the first quality signal; the metrics appendix gives the drift signal.
If injection proves redundant with a disciplined parent, the block
shrinks to the marker line - the infrastructure (store, provenance,
contract) survives either way, which is the actual asset.

---

## APPENDIX K - WORKED EXAMPLE: PENTEST SEAT BRIEF (engagement lessons)

Mandate: "test the client-x.com API surface for authz defects (IDOR)."

Query terms: api, authz, idor, client-x, burp.
Recall returns 4:

1. [RULE pentest-oracle 2026-11-01] scope gate active - every call
   matched against scope.yaml; read refusals, do not retry blind.
2. [GOTCHA scope-enforcer 2026-11-10] httpx probes every resolved
   subdomain - pin host lists or the gate blocks the sweep.
3. [RULE brain/cybersec 2026-11-02] IDOR testing: enumerate object ids
   across TWO accounts, never infer from one session.
4. [DECISION runlog-DR2 2026-11-02] LLM hypotheses need deterministic
   confirmation (sqlmap/autorize), never LLM-only claims.

Child JSON: lessons_followed [1,2,3,4]; method uses two accounts (3),
confirms sweep host list (2), runs under the gate (1), and validates
findings deterministically (3). The engagement report can cite, per
finding, the lesson context the seat operated under - provenance makes
the finding defense stronger, not just the spawn smarter.

---

## APPENDIX L - COMPOSITION WITH prompt-preflight (payload checks)

prompt-preflight validates dispatch payloads against its measured
failure classes (model-ref shapes, ${VAR} resolution, payload bloat,
excerpt-tail anchoring). The prior-art block interacts with three:

- payload size: the 40-line cap exists partly FOR preflight - blocks
  that bloat the brief trip the payload checks.
- excerpt anchoring: lesson lines are verbatim + provenance, so a
  truncated excerpt still anchors at the source line.
- depth-1 spawn: the block never instructs the child to spawn - it is
  evidence, structurally incapable of the re-delegation anti-pattern.

Pre-flight rejects briefs where the block broke the template - the two
skills compose at assembly time, before any spawn costs money.

---

## APPENDIX M - THE SELF-CONTAINED BRIEF RULE, PRECISELY

The subagents brief contract says briefs are SELF-CONTAINED (no session
history pasted). The prior-art block lives inside that rule because:

- it is TOP-N relevant, not the session; it does not carry the task,
  it annotates it.
- it is QUOTED + DATED, not narrated - the child can verify each line
  against its source (which a pasted history blob cannot offer).
- it carries no secrets, no credentials, no sibling negotiation - the
  brief contract rules apply to it as to the mandate itself.

The line between "self-contained brief" and "history dump" is exactly
the block cap: what the seat must know to execute lives in the brief
body; what the fleet already knows about the domain lives in the block.
When a lesson turns out to be REQUIRED task context (not prior art),
the brief was under-specified - fix the brief, not the lesson block.

---

## APPENDIX N - IMPLEMENTATION SKETCH (assembly-time hook, not per-call)

The recall runs at BRIEF ASSEMBLY (orchestrator-side), not as a PreToolUse
hook - the child never pays for its own recall:

```
assemble_brief(mandate, seat_type, run_dir):
  terms   = extract_terms(mandate, spec_r_keywords, class_hints)
  lessons = smem.recall(terms, kind=lesson, top=8)
  ranked  = rank(terms, recency, type)  # versioned ordering
  block   = compose(lessons[:5], cap=40, conflicts=dual)
  brief   = template(mandate, prior_art=block, out, deliverable, ...)
  preflight.validate(brief)
  return brief
```

The implementation is ~80 lines around the existing smem recall - the
skill is the CONTRACT (marker, verbatim, conflicts, JSON), not the
code. Any implementation satisfying Section 3 plus the caps is this
skill.

---

## APPENDIX N - GATES-LINE EXTENSION DETAILS

The slot values, exactly:

```
lesson-recall: fired (3 injected, 0 conflicts)
lesson-recall: fired (4 injected, 1 conflict -> child resolved: <short>)
lesson-recall: marker-on-empty (terms: <list>)
lesson-recall: grep-fallback (store unavailable; N lessons via grep)
lesson-recall: skipped: no spawn this turn
lesson-recall: skipped: store write failed (reported, not hidden)
```

Values never contain the pipe character. A spawn with NO recall slot and
NO skip line is an uncounted bypass - the audit counts it the same way
as every other gate bypass (the once-per-rollout V4 test exists to catch
the wiring gap, not to punish honest skips).

---

## APPENDIX N2 - THE VERBATIM RULE DEFENDED (why not summarize)

Summarizing a lesson is re-authoring it: the summarizer introduces its
own context, drops the constraint it does not understand, and produces
a NEW claim with the OLD lesson is authority. The measured confabulation
class is exactly that chain (summary drifts -> child trusts summary ->
fleet doctrine silently drifts). Verbatim + provenance means:

- the child sees the lesson as written (no telephone game),
- the audit can diff the injection against the store (tamper-evident),
- a reviewer can re-read the SOURCE, not trust the summary of it.

The cost is tokens (verbatim is longer); the cap (5/40) bounds it. The
exchange - bytes for fidelity - is the same trade every other gate in
this fleet makes, and it is the correct one: evidence is quoted,
never paraphrased.

---

## APPENDIX O - THE REVIEW INTEGRATION, CONCRETE

auto-review-gate reviewer packets gain one line when lessons were
injected:

```
prior_art: lessons 1,2,3 (ids) - check the child method against them
```

The reviewer then checks the three concrete failure shapes:

- RE-COMMITTED GOTCHA: the diff shows the documented trap re-implemented
  -> Major (the child contradicted evidence without stating it).
- INVENTED FACT: the child claims something the store disproves ->
  Major (the confabulation class, caught with its evidence).
- UNHONORED CONFLICT: lessons conflicted, child resolved in JSON,
  resolution is wrong -> Major with the resolution as the subject.

This makes the reviewer seat the enforcement point for the recall
contract - without review, the JSON field is decorative. The two skills
were designed as a pair: recall injects, review checks.

---

## APPENDIX P - WORKED EXAMPLE: WEB-DEV SEAT (niche lessons)

Mandate: "rebuild the cafe client landing hero section."
Query terms: cafe, landing, hero, design-system, deploy.
Recall returns 4:

1. [RULE brain/web-dev 2026-09-30] cafe sites must not look like SaaS
   (niche design DNA: warm palette, human photography).
2. [GOTCHA obs#201 2026-10-11] hero images: ship square assets;
   platforms mask avatars themselves (the 21.5% waste lesson).
3. [RULE design-md-system 2026-11-08] DESIGN.md before any build -
   source the system by niche, do not freehand.
4. [GOTCHA deploy 2026-11-12] client deploys: HTTP 200 probe + rollback
   pointer BEFORE the done-claim (blast-radius rule).

The child ships a warm-palette hero (1), square-source images (2),
after checking the DESIGN.md (3), and the deploy evidence lands in its
JSON (4). Each line is a fleet lesson that would otherwise be re-learned
by a fresh seat - the confabulation guard in its most visible form.

---

## APPENDIX Q - THE ONCE-PER-ROLLOUT WIRING TEST (V4, expanded)

The empty-marker test is the install gate. Procedure:

1. Craft a mandate with terms matching NOTHING in the store (nonsense
   domain: "recalibrate the flux capacitor budget").
2. Assemble a probe brief via the normal assembly path.
3. Open the brief: it MUST contain
   `[prior-art: none for <terms>]`.
4. Absence = the assembly is writing prior-art from memory (or not at
   all) - the wiring is missing; fix before any real dispatch.

The test costs one throwaway spawn. Without it, "no lessons found" and
"step never ran" are indistinguishable - the identical ambiguity that
made the D3 gate-format drift undetectable for weeks (a passing gate
that never fires is not a gate). The same arithmetic underlies review
V3, scope T10, and the converge bypass test: attack your own gate, once,
on purpose, and record the result.

---

## APPENDIX Q - THE ONE-PARAGRAPH DOCTRINE

Subagents exist to be de-correlated and focused - which makes them
amnesiac. The fleet compensates with handoffs (session-scoped) and the
brain (doctrine-level). The gap in between - the OPERATIONAL lessons,
recent and specific - was carried by whoever remembered to carry it.
This skill makes the carry mechanical: same store, deterministic query,
verbatim block, auditable contract. The fleet already paid for every
lesson in the store; paying for them again in re-derived bugs and
conflicting children is the one expense this skill removes.
