---
name: spec-clarify-gate
description: Front-load ambiguity resolution BEFORE planning on any build task - the ask-first pattern (Spec Kit /speckit.clarify, Kiro Quick Spec) formalized. Fires a structured, budgeted question round over the requirements, classifies each ambiguity (blocking / defaultable / deferrable), converts answered requirements into EARS-format acceptance criteria, and refuses to let planning start while a blocking ambiguity is open. Writes the clarified spec to a file that downstream skills (spec-analyze, spec-converge, auto-review-gate) consume as the requirements source of truth. Closes the biggest waste class on builds - building the wrong thing well. alwaysFire - fires on every build-task intake before planning; never on research, conversational, or read-only turns.
alwaysFire: true
---

# spec-clarify-gate - resolve ambiguity BEFORE planning

You are running **spec-clarify-gate**. A build task arrived (something will be
built, fixed, or changed). Before ANY planning, before ANY spawn, before even
a task list - you resolve the ambiguities in the request, with a budget, in
one round, and you freeze the result as the spec the rest of the chain
consumes.

**Why this exists (the measured failure class):** plans built on guessed
intentions are plans for the wrong thing. Every downstream phase - decompose,
spawn, build, review, converge - inherits the guess, and the cost of a wrong
assumption compounds per phase. The Spec Kit /speckit.clarify and Kiro
quick-spec patterns both measure the same thing: front-loading ambiguity
resolution is the cheapest phase of a build and the most frequently skipped.
The owner's standing rule (deep-research-when-unsure, guessing banned) is the
same doctrine pointed at intake instead of execution.

**What this skill is NOT:**

- NOT an interview simulator. Question budget is hard-capped (Section 3);
  most builds need 2-4 questions, not 20.
- NOT a requirements-writing skill. It resolves what is AMBIGUOUS, it does
  not invent scope. Features nobody asked for are spec-creep, and the
  out-of-scope list is written by this gate to stop them.
- NOT a blocker on research. Investigating to self-answer a question is
  preferred over asking (Section 2.4): ask ONLY what cannot be resolved by
  reading, searching, or the pinned conventions.

---

## 0) HARD RULES (survive compaction)

1. **No planning while a blocking ambiguity is open.** The gate is arithmetic:
   any requirement classified BLOCKING and unresolved = no tasklist, no spawn,
   no implementation. An honest question beats a confident wrong build.
2. **Self-answer first.** Every ambiguity is first tested against: (a) the
   existing codebase/brain (grep, recall), (b) the corpus index
   (session-memory-auto), (c) the stated conventions. Only what survives that
   becomes a question to the user. Questions cost round-trips; greps do not.
3. **Question budget: 5 blocking questions max per intake** (the measured
   attention span of a review round; Spec Kit caps clarify at 3-5 too). More
   than 5 means you did not do the classification work - batch, merge, or
   default them.
4. **Every question ships with a recommended default.** "X or Y? (I recommend
   Y because <one line>)" - a question without a recommendation is a
   permission-asking violation in disguise (the no-confirm standing rule).
   The user overrides the default by saying so; silence means default.
5. **Silence-does-not-block rule for non-blocking items.** Non-blocking
   ambiguities are resolved to the SAFEST reversible default, written to the
   spec as "assumed (reversible)", and surfaced in the plan - never asked.
6. **The spec file is the contract.** After clarification, the requirements +
   acceptance criteria + out-of-scope list are written to
   `<run-dir>/SPEC.md` (EARS format, Section 4). Every downstream consumer
   (auto-review-gate packet, spec-converge, spec-analyze) reads THAT file,
   not anyone's memory of the chat.
7. **Out-of-scope is written, not implied.** The spec names what this build
   does NOT do. An unwritten boundary is where scope creep lives.
8. **No invented requirements.** If the user said "make it faster" and you
   write "R5: p95 < 200ms", you invented a number - mark it as a proposed
   target and get it confirmed in the same question round, or write the
   requirement as the user's words with the measurement as an acceptance
   test to be agreed.
9. **One clarification round.** Batch all blocking questions into ONE
   message. Follow-up rounds only when an answer genuinely changes the
   question set (measured: each extra round costs a full turn-trip and the
   owner's patience).
10. **Trivial-ask escape hatch.** One-line fixes and conversational turns
    skip the gate (`clarify: skipped: trivial ask`). "In doubt, it is
    substantive" - a rebuild, migration, deploy, or anything touching shared
    state is never trivial.

---

## 1) WHEN IT FIRES

Always-fire on BUILD-TASK INTAKE - the moment a task will change artifacts:

| Intake signal | Gate action |
|---------------|-------------|
| "build / add / create / implement X" | full clarify round |
| "fix X" where the fix is not uniquely determined by the symptom | clarify the fix boundary (root-cause scope, blast radius) |
| "refactor / upgrade / redesign X" | clarify the success bar + what must not break |
| "deploy / migrate X" | clarify the rollback expectation + blast radius |
| "make X look/work better" (voice-dictated vibes) | clarify the observable success criteria (what would the user SEE working) |
| "continue X" with a handoff file | verify the handoff answers the blocking set; ask only the gaps |
| research / question / conversational | skip (`clarify: skipped: not a build`) |

Voice-dictation note (standing): the input is usually raw dictated intent.
Fix transcription, reconstruct the true intent, THEN run the gate on the
reconstructed intent - never ask the user to confirm your transcription of
their own words (that is the banned confirmation class).

---

## 2) THE AMBIGUITY TAXONOMY - classify before asking

Every ambiguity in the intake gets one of three classes. Classification is
the skill; asking is just the surface.

### 2.1 BLOCKING (must resolve before planning)

An ambiguity is BLOCKING when the plausible answers lead to MATERIALLY
DIFFERENT plans. Test: "if I plan for answer A and the truth is B, do I
delete work or add work?" Delete-work = blocking. Add-work = usually
defaultable.

Blocking examples:
- Target platform ("site for the VPS or static export?") - different stacks.
- Data authority ("does the existing DB stay the source of truth?") - different migrations.
- Definition of done ("9/10 on what dimension - speed, a11y, or visuals?").
- Scope of "all of them" (does 'all pages' include the archived ones?).

### 2.2 DEFAULTABLE (pick the safest reversible default, record it)

- Library choice within the established stack -> default to what the
  codebase already uses (reference-engine rule).
- Naming, file placement, error-message wording -> reversible defaults.
- Timeouts, retries, page sizes -> default to the codebase's existing
  constants.
- Anything the "simplest thing that works" rule already answers.

### 2.3 SITUATIONAL (depends on facts you can look up NOW)

- Existing behavior ("does the site already have auth?") -> grep, do not ask.
- Prior art ("did we already solve this?") -> memory recall (session-memory-
  auto), do not ask the user what they already told another session.
- Version/pricing/config drift -> web-verify, do not ask.

Classification rule of thumb: BLOCKING questions are the ones whose answers
change the FILE LIST or the ARCHITECTURE. If you cannot name which files
differ between the answers, it was not blocking.

---

## 3) THE QUESTION BUDGET AND THE RECOMMENDATION RULE

- Hard cap: 5 blocking questions per intake. If 6+ candidates exist, you
  failed to merge or default - do the work, then ask the survivors.
- Every question: stated as a question (not an option dump), carrying
  context ("this decides stack A vs B"), options with one marked
  **(Recommended)** + the one-line reason, and the cost of guessing wrong.
- Multiple-choice first, free-text last (voice-dictation users answer
  choices faster than prompts).
- The recommended default is what executes if the user never answers - the
  no-confirm rule means a silent user is a default-accepting user, so every
  recommended default must be SAFE to run unattended (if none is safe, that
  is itself a blocking escalation: pause for input, say exactly why).

---

## 4) EARS ACCEPTANCE CRITERIA - the format the spec file uses

EARS (Easy Approach to Requirements Syntax) - each acceptance criterion is
one of five patterns, verbatim prefixes:

```
WHEN <trigger> THE SYSTEM SHALL <response>
WHILE <state> THE SYSTEM SHALL <response>
WHERE <feature is enabled/condition> THE SYSTEM SHALL <response>
IF <condition> THEN THE SYSTEM SHALL <response>
THE SYSTEM SHALL <response>                      (unconditional)
```

Rules for writing them:

- One behavior per line; testable by one command or one interaction.
- No "should", no "properly", no "user-friendly" - those are not criteria,
  they are vibes. Replace with the observable ("returns HTTP 200 within
  500ms for the seeded dataset" not "handles load properly").
- Number them R1..Rn. The auto-review-gate coverage table uses these exact
  numbers. Downstream tools key on the R-numbers - renumbering mid-build
  breaks the chain.
- Each R carries a verification command (or "manual: <exact steps>"). An R
  with no verification is a wish and gets flagged at spec-analyze.

Example spec fragment:

```
R1: THE SYSTEM SHALL render the pricing table from data/prices.json.
    verify: pytest tests/test_pricing.py::test_render -q
R2: WHEN the API returns non-200 THE SYSTEM SHALL show the cached table
    with a "last updated" timestamp.
    verify: manual - kill the API, reload, observe banner
R3: WHERE a price is older than 30 days THE SYSTEM SHALL dim the cell.
    verify: unit test on the date logic
```

---

## 5) THE PROCEDURE (step by step)

**S1. Intake reconstruction.** Fix dictation errors; write the reconstructed
intent in one paragraph (internally or in the spec draft). Every explicit
requirement, constraint, output demand, tone - extracted as a checklist
BEFORE rewriting anything (the coverage anchor).

**S2. Ambiguity inventory.** List every ambiguity found, classified
(BLOCKING / DEFAULTABLE / SITUATIONAL). For SITUATIONAL: resolve immediately
(codebase grep, memory recall, web fetch) and strike them. For DEFAULTABLE:
pick the default now, record it with the one-line why. What remains is the
blocking set.

**S3. Budget check.** >5 blocking? Merge related ones, demote anything whose
wrong-guess cost is "add work" (not "delete work"), re-classify. Still >5?
Escalate honestly: state the task cannot be planned under the budget, list
the questions, stop. Never sneak a 7th question in.

**S4. The one question round.** Ask the blocking set, each with the
recommended option. Mark that this is a clarification round, not
permission-seeking (the work is already authorized; only the FACTS are
missing). Use the question tool when interactive; when the user is away
(autonomous runs), the recommended defaults become the working spec, logged
as `assumed: <Q> -> <default>` per question, and the build proceeds - with
the assumptions surfaced at the top of the plan so the user can veto on
return.

**S5. Write the SPEC.md.** Structure (fixed):

```
# SPEC - <task name> (<date>)
## Goal
<one paragraph, reconstructed intent>
## Requirements (EARS)
R1..Rn, each with verify
## Out of scope
- <named exclusions>
## Resolved defaults (no answer received)
- assumed: <question> -> <default> (reversible: <how>)
## Sources
- <what was read/fetched to ground the decisions>
## End-to-end check
<the single command or interaction that proves the whole thing>
```

**S6. Coverage gate (one pass).** Re-read the raw intake against the spec:
(a) every checklist item covered, (b) nothing invented, (c) nothing
over-extended. Fix in place. One pass. One line: `intent locked` or
`adjusted: <what>`.

**S7. Hand off.** Downstream (planner, subagents, auto-review-gate,
spec-converge) reads SPEC.md. The gates line reports
`clarify: fired (N blocking, M defaulted)`.

---

## 6) VERIFICATION COMMANDS

```
# V1. Spec exists and is complete
test -f <run-dir>/SPEC.md && grep -c "^R[0-9]" <run-dir>/SPEC.md
#    expect >= 1 R-number; a build with zero criteria has no gate

# V2. Every R has a verify line
grep -A1 "^R[0-9]" <run-dir>/SPEC.md | grep -c "verify:"
#    expect equal to the R count (R-numbers without verify = wishes)

# V3. Out of scope is written
grep -c "^- " <(sed -n '/## Out of scope/,/## Sources/p' <run-dir>/SPEC.md)
#    expect >= 1; empty boundary is a defect

# V4. EARS shape check (spot check)
grep "^R[0-9]" <run-dir>/SPEC.md | grep -vE "SHALL" && echo "EARS-violations" || echo "EARS-ok"

# V5. Downstream actually reads it (subagents briefs cite SPEC.md path)
grep -rl "SPEC.md" <tasklist or briefs dir>
```

V1-V3 are file checks (cheap, deterministic). V5 catches the drift where
briefs restate requirements from memory and diverge from the spec - the
exact failure spec-analyze exists for.

---

## 7) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| User answers 1 of 5 questions | over-budget round or unclear defaults | proceed on marked defaults for unanswered; do NOT re-ask; the defaults are logged and reversible |
| Spec written but plans diverge from it | downstream briefs didn't cite SPEC.md | V5 check; briefs must quote R-numbers, not paraphrase |
| "Ambiguity" was actually answerable by reading the code | skipped S2 situational resolution | strike it; asking what grep answers is the lazy-intake defect |
| Spec full of "should" and "properly" | EARS discipline skipped | rewrite the offending R-lines; run V4 |
| Blocking set empty but plan still guesses | mis-classified delete-work ambiguities as defaults | re-run the materiality test: different file lists = blocking |
| Scope grew mid-build past the spec | no out-of-scope list, or it was decorative | spec-converge catches the drift; backfill out-of-scope before the next build |
| Clarify round asked for PERMISSION not facts | habit violation | rephrase: every question carries a recommended default and states the work is already authorized |
| Voice input reconstructed wrong, spec built on the wrong intent | coverage gate skipped | S1 checklist + S6 re-read are mandatory even on trivial builds |

---

## 8) ANTI-PATTERNS

- THE INTERVIEW: serial single questions across turns. One batched round,
  budget-capped. Each extra round is a measured turn-cost with zero new facts.
- THE PERMISSION DISGUISE: "shall I use PostgreSQL or SQLite?" (permission)
  vs "PostgreSQL or SQLite? Recommend SQLite - the dep must stay
  zero-service" (clarification). The second is the only legal form.
- THE INVENTED SPEC: writing requirements the user never stated, however
  sensible. Record them as `proposed:` and get them into the question round
  or drop them.
- THE VANISHING OUT-OF-SCOPE: boundary only in someone's head. If it is not
  in SPEC.md, review cannot enforce it and scope creep wins by default.
- THE ETERNAL CLARIFY: treating clarify as a phase that can eat the whole
  session. The round is ONE message and the budget is FIVE. Builds ship.
- SPEC-FREE MICRO-BUILDS: skipping the file for a one-liner is fine
  (trivial skip); skipping it for anything touching 2+ files is not - the
  R-numbers are what make review and converge cheap.

---

## 9) WORKED EXAMPLES

### 9.1 The vibes-dictated rebuild

Intake (dictated): "rebuild the client landing page, it looks dated, make it
pop, you know, modern."

S2 classification:

- SITUATIONAL (resolve by reading): existing stack (Astro, per repo),
  brand colors (DESIGN.md exists), deploy target (the site is already on
  the VPS) - all struck.
- DEFAULTABLE: keep copy (flagged reversible), keep nav structure.
- BLOCKING: (1) "modern" for a law firm vs a cafe differs - which of the
  niches does the client site target and is the current brand direction
  kept? (2) is this a redesign of look-only, or is content/copy also on the
  table? (3) hard deadline or quality bar takes precedence for the
  iterate-until-9.5 loop?

Round: 3 questions, each with a recommendation (niche: keep current
restaurant niche per brain note; content: copy edits allowed, flagged;
deadline: quality bar wins unless stated). Two answered, one default ->
SPEC.md with 7 R-lines (perf budget, a11y AA, mobile, LCP, design-system
adherence, no regressions on existing routes, deploy evidence). Review and
converge later key on R1..R7. Total clarify cost: one message.

### 9.2 The fix with an unclear blast radius

Intake: "the deploy probe fails sometimes, fix it."

S2: reading the runbook shows three candidate root causes (stale serve
password, race on the port, provider 429). This is not an ambiguity in
REQUIREMENTS - it is an unknown in DIAGNOSIS. Misclassification risk: asking
the user "which is it?" is asking them to do the debugging. The gate
classifies: NOT blocking (the fix boundary is "make the probe reliable",
same file list under all three causes) -> proceed to diagnose with the
spec's R1: "WHEN the probe fails THE SYSTEM SHALL report the failing stage
with evidence" - the fix and the diagnosability land together.

Lesson recorded: clarify-gate does not front-load DIAGNOSIS questions, only
REQUIREMENT questions. Confusing the two produces interviews.

### 9.3 The autonomous run (user away)

Overnight goal loop, new task arrives at 02:00. S4 with no user: all blocking
questions take their recommended defaults, each logged:

```
assumed: "auth approach - extend Better Auth session vs new JWT service"
      -> extend Better Auth session (reversible: adapter swap, 1 file)
```

The build proceeds; SPEC.md carries the `assumed:` block; the morning report
leads with the assumptions so the owner can veto with the diff already
understood. This is the only legal form of proceeding on ambiguity in
unattended runs - marked, reversible, surfaced.

---

## 10) GATES-LINE CONTRACT

Slot: this skill feeds the canonical-14 line indirectly and reports in the
per-task extension:

```
clarify: fired (N blocking, M defaulted) | clarify: skipped: trivial ask | clarify: skipped: not a build
```

- `dim-completeness` cites the SPEC.md R-count when the task is a build.
- The auto-review-gate packet's requirements section IS the spec's EARS
  block; a review run without SPEC.md on a build task is a Major finding
  (spec exists, review did not read it).
- Interaction with subagents: briefs quote R-numbers; a brief that
  paraphrases instead of citing is rejected at wave 1 (coverage check).

Honest skip whys: `skipped: trivial ask` (one-line, no artifact),
`skipped: not a build` (research/read-only), `skipped: handoff-complete`
(continue-task with a handoff that answers the blocking set - verified, not
assumed: check the handoff answers each blocking question before skipping).

---

## 11) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| subagents | briefs carry R-numbers + the out-of-scope list verbatim; sibling OUT-lists align with the spec boundary |
| auto-review-gate | reviewer packet's requirements = SPEC.md EARS block; coverage table keys on R-numbers |
| spec-converge | converge re-assesses the BUILD against this SPEC.md at the end; the R-list is the checklist |
| spec-analyze | cross-artifact consistency check reads SPEC.md as the authority artifact |
| session-memory-auto | prior DECISIONS answer SITUATIONAL ambiguities (what we chose last time and why) |
| promptgod | the intake reconstruction + coverage gate is the same discipline applied to task intake |
| honestscore | "unmet requirement" findings cite R-numbers; the spec is what makes completeness measurable |

---

## 12) OPEN ITEMS (honest limits)

- The 5-question budget is a judgment constant, not a measured optimum;
  Spec Kit's own guidance is 3-5 and the campaign's user-response pattern
  supports the low end. Revisit if morning reports show default-veto rates
  above ~20%.
- EARS does not cover non-functional requirements (perf budgets, cost caps)
  gracefully; this skill piggybacks them as R-lines with numeric verify
  commands. A richer NFR syntax is a known gap; do not invent one silently.
- No automated ambiguity detector: classification is model judgment. The
  morning-report assumption-veto rate is the only quality signal until a
  labeled dataset exists (D10).

---

## APPENDIX A - QUESTION TEMPLATES (copy-paste, per ambiguity class)

### A.1 Blocking question format (the only legal form)

```
Q<n>. <the question, one sentence>
  Context: <what each answer changes - file list, stack, or DoD>
  A) <option>              (Recommended - <one-line reason>)
  B) <option>
  Cost of guessing wrong: <delete-work description, one line>
```

Filled example:

```
Q2. Does the client site keep its current copy during the redesign?
  Context: decides whether the copy pass is in scope (3 files) or out.
  A) Keep copy, design-only rebuild (Recommended - smallest reversible diff)
  B) Rewrite copy too (adds a copy-review pass; ~+2h)
  Cost of guessing wrong: A-guessed-but-wanted-B = new work added, not deleted
     -> this one is actually DEFAULTABLE by the add/delete test; only ask
        if the intake hints copy complaints.
```

Note the self-correction in the example - the act of WRITING the question
with the cost-of-guess line downgrades many "blocking" candidates to
defaults. That is the classification work happening on paper where it is
cheap.

### A.2 Default record format (no answer received)

```
- assumed: Q3 "deadline vs quality bar" -> quality bar wins
  reversible: yes - the iterate loop can be cut at any milestone
  surface: top of morning report
```

### A.3 Situational resolution record

```
- resolved-by-reading: "does the site already have auth?" ->
  grep -r "betterAuth" src/ -> 3 hits, session-based. No question asked.
```

Every resolved situational ambiguity gets one line in the spec's Sources
section - the provenance habit from session-memory-auto applies to intake
decisions too.

---

## APPENDIX B - FULL SPEC.md TEMPLATE (copy-paste)

```
# SPEC - <task name> (<YYYY-MM-DD>)
run: <run slug / session id>

## Goal
<one paragraph - the reconstructed intent in plain language. If this
paragraph cannot be written, the clarify round was not done.>

## Requirements (EARS)
R1: THE SYSTEM SHALL <...>
    verify: <command>
R2: WHEN <trigger> THE SYSTEM SHALL <response>
    verify: <command | manual: <exact steps>>
...

## Out of scope
- <named exclusion 1>
- <named exclusion 2>

## Resolved defaults (no answer received)
- assumed: <Q> -> <default> (reversible: <how>)

## Sources
- <file read / url fetched / memory recall id, one per decision>

## End-to-end check
<the single command or scripted interaction that proves the whole build>
```

Downstream consumers key on: the R-numbers (review coverage), the
out-of-scope list (converge drift detection), the end-to-end check
(verification-before-completion), and the Sources block (anti-fabrication).

---

## APPENDIX C - THE ONE-CARD OPERATING SUMMARY

```
INTAKE    reconstruct dictated intent -> extract the explicit checklist
INVENTORY classify every ambiguity: BLOCKING / DEFAULTABLE / SITUATIONAL
SITUATIONAL -> resolve NOW by reading/recall/fetch, strike it, log 1 line
DEFAULTABLE -> pick safest reversible default NOW, log with why
BLOCKING  -> max 5, each with a recommended option + cost-of-guess
ROUND     ONE batched message; silence = defaults execute (logged, reversible)
SPEC      SPEC.md: Goal / EARS R1..Rn + verify / Out-of-scope / defaults /
          Sources / end-to-end check
COVERAGE  re-read raw intake vs spec: covered, nothing invented, nothing
          over-extended -> "intent locked" or "adjusted: <what>"
## APPENDIX D - WORKED EXAMPLE: THE VETOED DEFAULT (what honest recovery looks like)

Overnight run assumed "SQLite, zero-service dep". Morning veto: "no, use the
existing Postgres - the data lives there." The veto is not a failure; it is
the system working:

1. The `assumed:` block made the assumption visible, so the veto is
   one sentence, not a fight.
2. The spec's `assumed:` entry marks reversibility ("adapter swap, 1 file")
   - the veto costs the swap, which is what the record predicted.
3. The morning report leads with the assumption block; the owner reads the
   veto against the SPEC.md diff, confirms, and the next R-lines inherit the
   corrected decision with the spec file updated the same turn.
4. The veto updates the brain: "for this project, DB = Postgres is a
   DECISION, not a default" - so the next intake's SITUATIONAL resolution
   finds it and the question never needs asking again.

The veto-rate metric (Section 12) exists to tune THIS loop: defaults that
survive = good defaults; defaults that get vetoed = either bad judgment
(tune the recommendation) or genuinely blocking items misclassified
(re-run the materiality test on that class).

## APPENDIX E - FAQ

**Q: Is asking a question "asking permission"?**
No - the banned pattern is asking permission to DO already-authorized work.
Asking for a FACT with a recommended default is the legal and required form.
The test: would the question disappear if the facts were known? Permission
asks disappear only if approval is granted.

**Q: The user answered with new requirements mid-round.**
Fold them into SPEC.md (new R-numbers continue from the last), strike the
questions they answered, and proceed. New requirements are not scope creep -
unauthorized scope growth is; requirements from the source are the point.

**Q: What if the task is 90% sure unambiguous?**
Run the inventory anyway; it takes minutes. The 10% is where rebuilds hide.
If the inventory genuinely finds zero blocking and zero defaultable items,
SPEC.md is still written (even a 3-line one) - the file is what converge and
review anchor to.

**Q: Multi-build sessions (several tasks in one turn)?**
One SPEC.md per build task, in the shared run dir with task suffixes
(SPEC-payment.md, SPEC-export.md). The gates line aggregates:
`clarify: fired (2 tasks: 3 blocking, 1 blocking; 2 defaulted)`.

**Q: The user answers a question I classified as defaultable?**
Their answer outranks the default. Update SPEC.md, re-run the coverage pass
(new answer can unblock new questions - but budget still applies).

---

## APPENDIX F - INTERACTION WITH VOICE-DICTATED INTAKE (standing)

The owner dictates most prompts (Serbian + English mix, transcription
artifacts common). Intake rules specific to that reality:

1. NEVER ask the user to confirm your transcription. Fix it silently; the
   reconstruction paragraph in SPEC.md IS the record of what you understood.
2. Diacritics and brand names: carry the user's spelling into the spec; if
   the codebase spells a name differently, the codebase wins for file paths
   and the user's spelling goes into user-facing copy decisions.
3. Numbers dictated as words ("around two seconds") become ranges in the
   draft R-line, not fake precision ("2.0s" is an invented number).
4. Mixed-language requirement: translate to the spec's working language
   (English), keep the original phrase in parentheses where the nuance
   matters - the reviewer reads the spec, not the audio.

5. It cannot be answered by a grep, a memory recall, or a web fetch.
6. It does not ask for permission to work (approval is standing).

The lint takes 2 minutes and prevents the two measured abuses: the lazy
intake (asking what grep answers) and the permission disguise. A round
where any question fails lint is rewritten before sending - the round trip
is the expensive resource; lint is free.

---
---

## APPENDIX G - QUESTION-QUALITY LINT (run before sending the round)

Every question in the batched round passes this 6-point check; a question
failing any line is rewritten or demoted:

1. It is a QUESTION (ends with the decision, not a monologue).
2. It names which files/architecture differ between answers (or it is not
   blocking - demote).
3. It carries a marked (Recommended) option with a one-line reason.
4. It states the cost of guessing wrong (delete-work vs add-work).
5. It cannot be answered by a grep, a memory recall, or a web fetch.
6. It does not ask for permission to work (approval is standing).

The lint takes 2 minutes and prevents the two measured abuses: the lazy
intake (asking what grep answers) and the permission disguise. A round
where any question fails lint is rewritten before sending - the round trip
is the expensive resource; lint is free.

## APPENDIX H - GRANULAR EXAMPLE: A 4-QUESTION ROUND, ANNOTATED

Intake: "add exports to the dump sorter - CSV at least, maybe the other
formats too, and the tool should remember settings."

S1 reconstruction: add an export feature to the Rabi C# credential-dump
toolchain (Dump Sorter); formats plural; persistence of settings implied.

S2 inventory:

- SITUATIONAL: existing export code (grep -> none), output directory
  conventions (repo rules file), C# version (10 per csproj) - all struck.
- DEFAULTABLE: async file I/O (project rule: async all I/O), error dialogs
  vs status bar (match existing UI pattern).
- BLOCKING candidates after merge:
  1. Formats: CSV only, or CSV+JSON+XLSX now? (file list differs; recommend
     CSV + JSON behind one exporter interface - XLSX needs a dep)
  2. "Remember" = settings file or full audit history? (different data
     model - delete-work risk)
  3. Large dumps: stream to file or load in memory? (architecture)
  4. Does export include the decrypted view or the raw dump? (correctness
     of the whole feature - blocking by the file-list test)

4 questions, all with recommendations, one round. Resulting R-lines cover
format interface, streaming, settings persistence, and the
decrypted-vs-raw rule. The exporter interface (IExporter) was pinned BEFORE
dispatch (shared assumption per subagents brief contract).

The example demonstrates the budget in practice: four questions happened to
The example demonstrates the budget in practice: four questions happened to
fit without demotion, because the add/delete test demoted "CSV+JSON or just
CSV?" to defaultable (interface accepts new formats by adding a class -
add-work only).

---

## APPENDIX I - DOWNSTREAM CONSUMER CONTRACT (what each reader takes)

SPEC.md is consumed by four skills; each consumes a specific block and each
has a rejection condition:

| Consumer | Reads | Rejects the spec when |
|----------|-------|------------------------|
| planner / subagents | Requirements (R-numbers) + out-of-scope | briefs would need to paraphrase (they cite R-numbers instead) |
| auto-review-gate | EARS block (as the coverage table source) | any R lacks a verify line |
| spec-converge | Requirements + end-to-end check | the end-to-end check is not runnable |
| spec-analyze | Whole file (authority artifact) | two R's contradict, or an R contradicts the goal paragraph |
| spec-converge (append) | Resolved defaults | a default is not marked reversible |

The contract makes the spec the single requirements authority: when review,
converge, and analyze disagree about what was asked, SPEC.md wins - and a
change to requirements mid-build is a spec EDIT (versioned in the file's
run line), never a verbal agreement.

Spec version rule: edits append a line `## Changelog - <date>: <what and
why>`; the R-numbers are never renumbered (consumers key on them); a
removed R is marked `(removed <date>: why)`, not deleted - converge and
review must still see that R4 once existed, or their coverage tables
silently lose a row.

---

## APPENDIX J - MEASUREMENT (how the gate knows it works)

Recorded per intake, one line in the runlog:

```
clarify | <date> | task <slug> | blocking N (answered N, defaulted M) |
rounds <n> | veto-rate <vetoed/assumed> | spec <path> | R-count <n>
```

Tracked signals and their thresholds:

- **default-veto rate** (morning report vetoes / assumptions made): >20%
  means the recommendation quality is off - tune the recommendation rule or
  raise the question budget for that task class.
- **rounds per intake**: target 1.0; two rounds = the S4 batching failed;
  three = the taxonomy was misapplied (a blocking question was hidden
  inside an answer).
- **clarify-to-plan latency**: the whole gate should cost minutes, not a
  session. >30 min wall-clock on clarify = the inventory step is eating
  itself; demote harder.
- **spec drift events** (requirements changed mid-build): each one is
  either a missed blocking ambiguity (tune the classifier) or a genuine
  late requirement (fine - the changelog records it).

These feed the same calibration pipeline as the review metrics (auto-review-
gate Appendix G) - one labeled dataset for both gates.

---

## APPENDIX K - THE THREE CLASSIFICATIONS, FULLY WORKED

One intake, all three classes exercised, showing the classifier's reasoning
in full. Intake: "our API is slow, fix it, and while you are there add rate
limiting."

- "slow" -> SITUATIONAL: profile first (ideas code-performance rule). No
  question. The R-line becomes "R1: WHEN the profile is re-run THE SYSTEM
  SHALL show p95 <= <target>" where the target is proposed for confirmation,
  not invented (Rule 0.8).
- "our API" -> resolved by reading: one FastAPI service + one worker; the
  ambiguity "which API" dies to a grep.
- "add rate limiting" -> the limit VALUE is DEFAULTABLE (start conservative,
  config-exposed, reversible); the SCOPE (which endpoints) is BLOCKING if
  "our API" spans a public and an admin surface with different owners -
  file-list test says: admin auth path differs -> blocking, 1 question.
- "while you are at it" -> the scope-creep trigger: the out-of-scope list
  explicitly names what "while you are there" does NOT include (no schema
  changes, no new endpoints, no client changes) unless confirmed.

Net: ONE blocking question (admin surface in or out), one proposed target
(p95), one decorative-creep boundary written. The intake LOOKED like it
needed a long interview; the taxonomy reduced it to one question. That is
the skill working.

---

## APPENDIX L - COMPOSITION WITH PLANNING (what clarify hands the planner)

The gate ends where planning begins. The handoff contract:

```
clarify outputs:
  SPEC.md                    <- the contract (R-numbers, boundary, e2e check)
  pinned assumptions         <- names/interfaces/conventions decided BEFORE
                                dispatch (subagents brief contract item 4)
  open (non-blocking) notes  <- defaults + risks, for the plan's risk section
planner receives EXACTLY those three, and:
  - plans only the in-scope surface
  - pins the shared assumptions into every brief (never re-negotiated by
    siblings - subagents rule)
  - maps each plan unit to R-numbers it fulfills (coverage in, coverage out)
```

A planner receiving a clarify-less build task (legacy flow) reconstructs the
minimum spec itself (self-serve S1-S6) and records
`clarify: self-serve (N defaults)` in the gates line - better than nothing,
marked as degraded. The chain should make this path rare: intake always
passes through this gate first.

---

## APPENDIX M - THE ESCALATION PATH (when defaults are NOT safe)

Rule 3's corollary states it; this is the exact procedure when no
recommended default is safe to run unattended:

1. STOP the build phase (not the session): write SPEC.md with the blocking
   set marked `UNRESOLVED - needs owner input`, and do every safe partial
   first (recon, brain reads, profile) so the pause costs nothing.
2. Surface ONE message with the questions and exactly what each unlocks -
   per the owner-touches-only-three-things rule, name only the single
   decision the owner must make, and do everything else yourself first.
3. Arm the watchdog on the pause (session-monitor-auto): a blocked intake
   that sits >2 beats with no answer and no partial work is a stall the
   monitor flags. A pause must be productive on everything that does not
   need the answer.
4. On the answer: update SPEC.md (changelog line), strike the question,
   proceed at full speed.

This is the only legal mid-build pause besides a manual-credential input.
"Waiting on requirements" without the SPEC.md `UNRESOLVED` marking is not a
pause - it is a hidden stall, and the watchdog treats it as one.

---

## APPENDIX N - ESCALATION EXAMPLE (the pause that stayed productive)

Intake: "migrate the client sites to the new registrar" arriving at 01:30 in
an autonomous run. The blocking set includes one genuinely unsafe default:
transferring domains involves registrar-side auth codes (owner-owned
credential, the standing exception).

The gate: writes SPEC.md (R1: no site downtime; R2: DNS cutover verified
with real probes; R3: old registrar deactivated only after 48h of stable
resolves), does every safe partial (inventory of the 7 domains, TTL
lowering plan, DNS provider scripts prepared, rollback notes), marks the
auth-code step `UNRESOLVED - needs owner input`, and surfaces ONE line:
"need the registrar auth codes for X - only you can supply; everything else
is staged." The watchdog watches the pause; the morning report shows staged
work and a single named input. When the codes land, execution is a 20-minute
procedure that was already written.

Contrast with the anti-pattern: a session that "waits on requirements"
while doing nothing, with nothing marked - the owner returns to a hidden
stall wearing the costume of diligence.

