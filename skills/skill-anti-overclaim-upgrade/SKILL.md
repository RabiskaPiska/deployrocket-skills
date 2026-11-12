---
name: skill-anti-overclaim-upgrade
description: Detects overclaiming language before done-claims are allowed. 60 blocked phrases in 3 categories, formal detection algorithm with question-form and context-window exclusions, PASS/BLOCK output formats, PASS-WITH-NOTE tie-break, and self-test suite. Claim gate #2 of 4 - fires after skill-ground-truth-harness, before skill-paperclip-ledger-guard. Supersedes the base anti-overclaim phrase scan for gate ordering; the alias twin anti-overclaim remains active as the base layer (gate #4) - either id fires the same stack.
alwaysFire: true
---

# Anti-Overclaim Upgrade - 60-Phrase Blocker

## SUPERSESSION AND ALIAS MAP

This skill is the LEAD in the overclaim-detection pair. Relationship to the
other skills, fixed here so no session has to re-derive it:

- `skill-anti-overclaim-upgrade` (THIS SKILL) - Gate 2. Mechanical 60-phrase
  detection with the context-window and question-form rules. The lead.
- `anti-overclaim` - the base layer, Gate 4. Its evidence-format rules (E1-E5)
  remain authoritative for what counts as evidence; its phrase table is
  superseded by this skill's 60-phrase list for DETECTION purposes. The alias
  map may fire either id; both route into the same four-gate chain, so firing
  either is correct and complete.
- Order: skill-ground-truth-harness (Gate 1) -> THIS (Gate 2) ->
  skill-paperclip-ledger-guard (Gate 3) -> anti-overclaim base (Gate 4).

Consequence: adding new blocked phrases happens HERE. The base skill's phrase
list is frozen reference material, not the live detector.

## SECTION 1 - WHEN TO FIRE

Fire on EVERY response that contains a done-claim, fixed-claim, shipped-claim,
or deployed-claim.

A **done-claim context** means: the phrase appears near (within ~100 characters
of) any of these context words:

> done, ship, deploy, finish, complete, fixed, working, pass, verified,
> confirmed, live, active

**Fire on** assertions like:

- "The deployment is complete."
- "This should work now."
- "I've set it up and it's running."
- "Everything looks correct."

**Do NOT fire on:**

- Questions: "Does this look correct?" - NOT blocked
- Hypotheticals: "In theory, this would work if the config is correct." -
  NOT blocked
- Descriptions of requirements: "The function should work by calling X." -
  NOT blocked (describing intent, not asserting a result)

If the blocked phrase appears inside a sentence ending in `?`, skip the block
entirely.

## SECTION 2 - THE 60-PHRASE LIST

### Category A - Soft Done-Claims (25 phrases)

These assert completion without attaching any evidence:

1. `done`
2. `complete`
3. `completed`
4. `finished`
5. `deployed`
6. `shipped`
7. `working`
8. `fixed`
9. `resolved`
10. `implemented`
11. `ready`
12. `confirmed`
13. `verified`
14. `tested`
15. `validated`
16. `passing`
17. `successful`
18. `live`
19. `active`
20. `enabled`
21. `configured`
22. `set up`
23. `installed`
24. `running`
25. `operational`

### Category B - Confidence Overclaims (20 phrases)

These assert certainty or near-certainty without proof:

1. `should work`
2. `looks correct`
3. `probably fine`
4. `close enough`
5. `almost done`
6. `nearly there`
7. `looks good`
8. `seems right`
9. `appears to work`
10. `should be`
11. `likely works`
12. `ought to work`
13. `it must be`
14. `clearly works`
15. `obviously correct`
16. `certainly works`
17. `definitely done`
18. `without doubt`
19. `this will work`
20. `no issues`

### Category C - Untestable Claims (15 phrases)

These make assertions while explicitly or implicitly blocking verification:

1. `cannot test but`
2. `trust me`
3. `obviously works`
4. `no need to verify`
5. `it's clear that`
6. `logically correct`
7. `must be working`
8. `theoretically correct`
9. `in principle works`
10. `should be fine`
11. `I believe this works`
12. `this ought to pass`
13. `logically sound`
14. `no way this fails`
15. `this is definitely right`

**Total phrase count: 25 + 20 + 15 = 60** (count is load-bearing: the self-test
in SECTION 9 asserts it).

## SECTION 2A - WHY EACH CATEGORY EXISTS

- Category A phrases are legitimate words that become overclaims when used as
  the SOLE support for a completion assertion. The phrase list is intentionally
  broad; the context-window rule is what makes it precise.
- Category B phrases smuggle in an unstated probability ("should", "looks",
  "probably"). The replacement is always a measurement or "untested".
- Category C phrases are the most dangerous: they PRE-EMPT verification. Any
  sentence that argues checking is unnecessary is itself the violation.

Historical anchors (each rule below is the scar of a real failure):

- "A self-score of 7.7 against blind adversarial passes of 6.0/3.5/5.0" - the
  self scored its own prose; the phrase "looks correct" survived six read-
  throughs and zero commands. Category B exists because re-reading is not
  checking.
- A fix once INTRODUCED the exact defect it guarded against. "Fixed" plus
  "should work" is the sound of that failure. Categories A and B both apply.
- "Could not measure" written as if it were a pass result. Category C exists
  to make that sentence impossible to emit.

## SECTION 3 - DETECTION ALGORITHM

All matching is case-insensitive. Phrase matching uses substring search (not
word-boundary) to catch embedded uses like "it's completed" matching
`completed`.

```python
CAT_A = [
    "done", "complete", "completed", "finished", "deployed", "shipped",
    "working", "fixed", "resolved", "implemented", "ready", "confirmed",
    "verified", "tested", "validated", "passing", "successful", "live",
    "active", "enabled", "configured", "set up", "installed", "running",
    "operational"
]

CAT_B = [
    "should work", "looks correct", "probably fine", "close enough",
    "almost done", "nearly there", "looks good", "seems right",
    "appears to work", "should be", "likely works", "ought to work",
    "it must be", "clearly works", "obviously correct", "certainly works",
    "definitely done", "without doubt", "this will work", "no issues"
]

CAT_C = [
    "cannot test but", "trust me", "obviously works", "no need to verify",
    "it's clear that", "logically correct", "must be working",
    "theoretically correct", "in principle works", "should be fine",
    "I believe this works", "this ought to pass", "logically sound",
    "no way this fails", "this is definitely right"
]

DONE_CONTEXT_WORDS = [
    "done", "ship", "deploy", "finish", "complete",
    "fixed", "working", "pass", "verified", "confirmed",
    "live", "active"
]


def is_question_sentence(text: str, phrase_idx: int) -> bool:
    """Return True if the sentence containing phrase_idx ends with '?'."""
    start = text.rfind('.', 0, phrase_idx)
    start = max(start, text.rfind('!', 0, phrase_idx))
    start = max(start, text.rfind('\n', 0, phrase_idx))
    end = text.find('.', phrase_idx)
    end_q = text.find('?', phrase_idx)
    end_e = text.find('!', phrase_idx)
    terminators = [x for x in [end, end_q, end_e] if x != -1]
    if not terminators:
        return False
    nearest_end = min(terminators)
    return text[nearest_end] == '?'


def check_overclaim(response_text: str) -> dict:
    response_lower = response_text.lower()

    for category_name, phrases in [("A", CAT_A), ("B", CAT_B), ("C", CAT_C)]:
        for phrase in phrases:
            phrase_lower = phrase.lower()
            idx = response_lower.find(phrase_lower)
            while idx != -1:
                if is_question_sentence(response_lower, idx):
                    idx = response_lower.find(phrase_lower, idx + 1)
                    continue

                window_start = max(0, idx - 100)
                window_end = min(len(response_lower),
                                 idx + len(phrase_lower) + 100)
                surrounding = response_lower[window_start:window_end]

                if any(word in surrounding for word in DONE_CONTEXT_WORDS):
                    return {
                        "result": "BLOCK",
                        "phrase": phrase,
                        "category": category_name,
                        "surrounding": response_text[window_start:window_end],
                    }

                idx = response_lower.find(phrase_lower, idx + 1)

    return {"result": "PASS"}
```

### Edge-case rules

| Situation | Behaviour |
|---|---|
| Phrase inside a `?` sentence | Skip - not a done-claim |
| Phrase in a backtick-fenced code block | Still matched - the agent may be describing its own output |
| Multiple triggers in one response | Report the FIRST hit; block immediately; do not scan further |
| Phrase in a quoted user message | Still matched - the agent quoted it approvingly |
| Hypothetical with explicit "in theory" prefix AND no context word nearby | Skipped by the context-window check |
| Phrase inside a URL or file path | Still matched; rephrase paths in prose, not fence the claim |
| Phrase inside the evidence output being quoted | PASS-WITH-NOTE rule (Section 8, Example 6) |

## SECTION 3 - FALSE-POSITIVE CONTROLS

The blocker's failure mode is nagging: if it blocks honest reporting often
enough, agents start avoiding the word "done" instead of attaching evidence,
which is worse. Controls, in force:

1. Question-form exclusion (sentences ending in `?` are never claims).
2. Context-window requirement: no block without a done-context word nearby.
3. Hypothetical carve-out: "in theory" + no context word = no block.
4. Requirement-description carve-out: "should work by calling X" describes
   intent; the context-window rule is what saves it - verify the window
   before firing.
5. Evidence co-presence: if real pasted output sits in the same response,
   the PASS-WITH-NOTE tie-break applies (Section 8) rather than a hard block.

When a false positive DOES occur: it is fixed by refining the CONTEXT WINDOW
or adding an exclusion, never by deleting the phrase. Deleted phrases are how
the gate goes quietly blind.

## SECTION 4 - BLOCK OUTPUT FORMAT

When a phrase triggers a block, output exactly this structure (fill in the
bracketed fields; ASCII only - no box-drawing characters, they break some
terminals and fail ASCII gates):

```
==========================================================
  ANTI-OVERCLAIM BLOCK DETECTED (Gate #2)
============================================================

Phrase   : "[exact phrase found]"
Category : [A - Soft Done-Claim | B - Confidence Overclaim | C - Untestable Claim]
Found in : "[quote the surrounding 1-2 sentences verbatim]"

------------------------------------------------------------
 Execution is PAUSED. Provide ONE of the following to continue:
------------------------------------------------------------

  (a) Command output pasted directly
      -> paste the raw terminal/tool result, not a description of it

  (b) File read-back showing expected content
      -> paste the actual file contents as returned by the Read tool,
        not "I saved it" or "it should contain"

  (c) Test run output showing pass/fail
      -> paste the full test runner output including exit code

  (d) Explicit "untested" acknowledgment:
      -> "untested - exact command to verify: [command the user can run]"

------------------------------------------------------------
 Gate 2 status: BLOCKED - skill-paperclip-ledger-guard will NOT
 receive this response until evidence is provided and re-scan passes.
------------------------------------------------------------
```

**After evidence is provided**, re-run `check_overclaim()` on the revised
response. If the revised response no longer contains a blocked phrase in a
done-claim context, issue the PASS output (Section 4) and forward to Gate 3.

## SECTION 3 - PASS OUTPUT FORMAT

When no blocked phrase is found in a done-claim context (or after evidence
resolves a prior block):

```
============================================================
  ANTI-OVERCLAIM: PASS (Gate #2)
============================================================
No blocked phrases detected in done-claim context.
Forwarding to Gate 3: skill-paperclip-ledger-guard
```

PASS-WITH-NOTE (evidence pasted, phrase used descriptively alongside it):

```
ANTI-OVERCLAIM: PASS-WITH-NOTE - phrase "[phrase]" (Cat-A) present but
accompanied by pasted tool output (evidence accepted by context).
Forwarding to Gate 3: skill-paperclip-ledger-guard
```

## SECTION 4 - INTEGRATION CHAIN

This skill occupies Gate 2 in the four-gate claim-verification chain:

```
Response containing a done / fixed / shipped / deployed claim
        |
  [Gate 1] skill-ground-truth-harness
        |    Provides tool output / evidence.
        |    Requires at least one verifiable side-effect before passing on.
        |
  [Gate 2] skill-anti-overclaim-upgrade    <-- THIS SKILL
        |    Scans response text. BLOCK -> pause + request evidence,
        |    then re-scan.
        |
  [Gate 3] skill-paperclip-ledger-guard
        |    HMAC bundle sign-off. Checks ledger integrity.
        |
  [Gate 4] anti-overclaim (base layer)     Final check / evidence block.
```

**Data flow:**

| Step | Actor | Input | Output |
|---|---|---|---|
| 1 | skill-ground-truth-harness | raw response | response + evidence bundle |
| 2 | THIS SKILL | response + evidence bundle | PASS or BLOCK result |
| 3 | skill-paperclip-ledger-guard | PASS result + evidence bundle | HMAC-signed bundle |
| 4 | anti-overclaim (base) | signed bundle | final PASS / FAIL |

**On BLOCK:** execution stops at Gate 2. The response returns to the agent
with the BLOCK output (Section 4 above). The agent supplies evidence. After
evidence is attached, `check_overclaim()` re-runs on the updated text. Only a
clean re-scan advances to Gate 3.

**Gate 2 never modifies response content.** It inspects and routes only.

## SECTION 5 - GATE LINE CONTRIBUTION

Every turn that passes Gate 2 appends to the claim-gates line:

**On PASS:**

```
claim-gates: ... | anti-overclaim-upgrade=PASS
```

**On BLOCK (before resolution):**

```
claim-gates: ... | anti-overclaim-upgrade=BLOCK:["phrase" Category A]
```

**On BLOCK resolved to PASS (evidence provided, re-scan clean):**

```
claim-gates: ... | anti-overclaim-upgrade=PASS(was-BLOCK:["phrase" Cat-A, resolved)
```

The full gate line across all four gates:

```
claim-gates: ground-truth-harness=PASS | anti-overclaim-upgrade=PASS | paperclip-ledger-guard=SIGNED:bundle-verified | anti-overclaim=PASS
```

## SECTION 6 - EXAMPLES

### Example 1 - PASS (evidence provided alongside claim)

**Input response:**

```
The migration script ran successfully. Here is the output:
  $ python migrate.py
  Applying migration 0003_add_index... OK
  All migrations applied. Exit 0.
```

**Gate 2 result:** `successfully` and `OK` are not in the 60-phrase list.
Context word `applied` present but no blocked phrase triggers.

```
ANTI-OVERCLAIM: PASS - no blocked phrases detected in done-claim context
```

### Example 2 - BLOCK (Category A, bare done-claim)

**Input response:**

```
The deployment is complete.
```

**Gate 2 result:** phrase `complete` (Category A); context word `deployment`
matches `deploy`; not a question sentence.

```
ANTI-OVERCLAIM BLOCK DETECTED (Gate #2)

Phrase   : "complete" (Category A - Soft Done-Claim)
Found in : "The deployment is complete."

Execution is PAUSED. Provide ONE of:
  (a) Command output pasted directly
  (b) File read-back
  (c) Test run output with exit code
  (d) "untested - exact command to verify: [command]"
```

### Example 3 - BLOCK (Category B, confidence overclaim)

**Input response:**

```
This should work now that the config has been updated.
```

**Gate 2 result:** phrase `should work` (Category B); context word
`config`/`updated` in window; not a question.

```
ANTI-OVERCLAIM BLOCK DETECTED (Gate #2)

Phrase   : "should work" (Category B - Confidence Overclaim)
Found in : "This should work now that the config has been updated."

Execution is PAUSED. Provide evidence.
```

### Example 4 - PASS (question-form exclusion)

**Input response:**

```
Does this look correct to you?
```

**Gate 2 result:** `looks correct` not found (text has `look correct` - no
substring match); and the sentence ends in `?`, so the question-form
exclusion would apply regardless.

```
ANTI-OVERCLAIM: PASS - no blocked phrases detected in done-claim context
```

### Example 5 - BLOCK (Category C, untestable claim)

**Input response:**

```
I cannot test but the logic is clearly sound and the function is verified.
```

**Gate 2 result:** first hit `cannot test but` (Category C); context word
`verified` nearby; not a question.

```
ANTI-OVERCLAIM BLOCK DETECTED (Gate #2)

Phrase   : "cannot test but" (Category C - Untestable Claim)
Found in : "I cannot test but the logic is clearly sound and the function is verified."

Execution is PAUSED. Provide evidence or state:
  "untested - exact command to verify: [command]"
```

### Example 6 - PASS-WITH-NOTE (evidence resolves the block)

**Agent's revised response after the block in Example 2:**

```
Deployment output:
  $ kubectl rollout status deployment/api
  deployment "api" successfully rolled out
  Exit 0.
The rollout finished without errors (exit 0 above).
```

**Gate 2 re-scan:** `finished` (Category A) is still present, but it is used
descriptively alongside pasted tool output. Tie-break rule applies:

- If evidence is pasted AND the phrase is used descriptively alongside the
  evidence (not as the sole claim), Gate 2 notes the phrase but marks it
  PASS-WITH-NOTE, forwarding to Gate 3 with a flag.

```
ANTI-OVERCLAIM: PASS-WITH-NOTE - phrase "finished" (Cat-A) present but
accompanied by pasted tool output (evidence accepted by context).
Forwarding to Gate 3: skill-paperclip-ledger-guard
claim-gates: ... | anti-overclaim-upgrade=PASS(was-BLOCK:"complete" Cat-A, resolved)
```

## SECTION 7 - RESOLUTION LIFECYCLE (state machine)

A blocked claim moves through exactly these states - nothing skipped, nothing
repeated:

```
SCAN -> HIT? -> BLOCK(response returned to agent)
                  |
                  v
            EVIDENCE PROVIDED (a/b/c/d from Section 4)
                  |
                  v
            RE-SCAN(updated response text)
                  |
        +---------+----------+
        |                    |
      clean               still dirty
        |                    |
        v                    v
  PASS-WITH-NOTE        BLOCK again (same phrase, now WITH
        |               evidence attached -> PASS-WITH-NOTE if
        v               evidence is real; else remain BLOCKED)
  Gate 3
```

Rules:

- Re-scan always runs on the UPDATED response text, never the original.
- The same phrase may block at most twice per turn; the second block must
  name what evidence is still missing, explicitly.
- Three blocks on one claim = escalation: the claim is downgraded to
  UNTESTED and the task re-queued with the missing-verification note.
- The lifecycle never edits the agent's text; it only pauses, requests,
  and re-scans.

## SECTION 8 - SELF-TEST (the gate tests its own list)

The phrase list and detector have a self-test that must pass after ANY edit
to this skill. Run it before relying on a modified list:

```python
def test_self():
    # 1. Count integrity: 25 + 20 + 15 = 60
    assert len(CAT_A) == 25 and len(CAT_B) == 20 and len(CAT_C) == 15

    # 2. No phrase appears in two categories (dedup)
    all_phrases = CAT_A + CAT_B + CAT_C
    assert len(set(p.lower() for p in all_phrases)) == 60

    # 3. Canonical blocks
    assert check_overclaim("The deployment is complete.")["result"] == "BLOCK"
    assert check_overclaim("This should work now.")["result"] == "BLOCK"
    assert check_overclaim(
        "I cannot test but the logic is verified.")["result"] == "BLOCK"

    # 4. Canonical passes
    assert check_overclaim("Does this look correct?")["result"] == "PASS"
    assert check_overclaim("The output above shows exit 0.")["result"] == "PASS"

    # 5. Context-window rule: phrase with NO context word nearby = PASS
    assert check_overclaim(
        "The recipe says the dough is done resting overnight."
    )["result"] == "PASS"

    # 6. Question-form exclusion
    assert check_overclaim("Is it done?")["result"] == "PASS"
```

Failure of any assertion blocks any done-claim in the session that modified
the list until fixed. The self-test exists because a phrase list edited
without verification has, in practice, silently emptied itself.

## SECTION 9 - EXTENDING THE LIST (process, not vibes)

New phrases enter by incident, not by imagination:

1. A false-done occurs (or nearly occurs) in a real session.
2. The exact wording is quoted and dated in the incident note.
3. The phrase is added to the correct category with a one-line "why it
   fails" comment.
4. The self-test in Section 8 is re-run and pasted.
5. The base anti-overclaim skill's phrase table is NOT edited - this skill
   is the single live list (see Alias Map note at top).

Retirement is allowed only with a recorded false-positive incident showing
the phrase blocks legitimate usage with no available alternative phrasing.

## SECTION 10 - FAQ

**Q: Does this block honest uncertainty like "should be done by now"?**
Category B targets certainty claims, not schedules. "Should be done by
Friday" has no done-claim context word adjacent to an assertion of
completion - no block. "This should work" asserts behavior without
measurement - block. The context window is what disambiguates.

**Q: What if the evidence contradicts the claim?**
Out of scope for Gate 2 (it is a prose scanner). The contradiction is a
defect for proven-working's contradiction-handling protocol: stop, re-run
both checks, then escalate.

**Q: What if the agent games the detector with synonyms?**
Then Gate 1 (ground-truth harness) still requires pasted evidence for the
done-claim, and Gate 4 requires the evidence block. The phrase list is a
tripwire, not the foundation - the evidence requirement is.

**Q: Are code blocks scanned?**
Yes - the agent may be describing its own output in them, and "done" inside
a fenced block next to a context word has historically been a real overclaim
(the agent narrating an unrun command's expected output).

**Q: Where does this sit relative to honestscore?**
Upstream. honestscore judges quality on a rubric; this gate guarantees the
response does not claim completion without evidence before quality is even
discussed. Skipping the gate to reach the score is itself an overclaim.

## SECTION 11 - TIE-BREAK DECISION TABLE

| Situation | Verdict |
|---|---|
| Phrase + context word + no evidence anywhere in response | BLOCK |
| Phrase + context word + real pasted tool output in response | PASS-WITH-NOTE |
| Phrase + context word, sentence is a question | no block (question rule) |
| Phrase + "in theory" prefix, no done-context word in window | no block (context rule) |
| Phrase in a fenced code block adjacent to a completion claim | block (narrated output) |
| Phrase quoting the USER's message approvingly | block |
| Phrase in an unrelated sense (recipe, game, doc quote) | context-window rule decides |
| Evidence pasted AFTER a block, phrase still present | PASS-WITH-NOTE, gate line records the resolution |
| Evidence contradicted by the phrase ("tested" with no run shown) | BLOCK stays; evidence must be the actual output |

The table is exhaustive by design: new situations join the table via
Section 9's incident process instead of being improvised per turn.

## SECTION 11 - FALSE-POSITIVE LEDGER

Wrongly-blocked responses are recorded with their resolution. The ledger
keeps the detector precise in both directions (misses AND false alarms):

| Date | Phrase | Context | Wrongly blocked? | Resolution |
|---|---|---|---|---|
| 2026-11-08 | complete | "the emoji set is complete" (list of 10 pasted) | yes | precision rule: pasted-evidence presence converts a block to a note |
| (none since) | | | | |

Rules:

- A false positive is fixed by ADDING a precision rule (evidence presence,
  question form, context window) - never by deleting the phrase.
- Every precision rule gets a self-test case in Section 8 so it cannot
  silently regress.
- Precision rules stack in a fixed order: question-form first, then
  hypothetical prefix, then evidence-presence tie-break, then block.

## SECTION 12 - ESCALATION MATRIX

| Event | Action |
|---|---|
| 1st BLOCK on a claim | pause; request evidence (a/b/c/d) |
| evidence provided, re-scan clean | PASS (or PASS-WITH-NOTE if a flagged phrase remains beside the evidence) |
| re-scan still BLOCKs with evidence present | show which phrase fired; agent may rephrase or attach evidence differently |
| 2nd BLOCK on the same claim | pause; explicitly name what evidence is still missing |
| 3rd BLOCK on the same claim | claim downgraded to UNTESTED; task re-queued with the missing verification named |
| detector unavailable/misfires | gate line reads anti-overclaim-upgrade=skipped:detector-error; the claim still cannot pass without Gate 1 evidence |

## SECTION 13 - FULL WORKED TRANSCRIPT (block, then resolution)

**Turn 1 - agent emits a bare claim:**

```
Agent: Migrated all three services to the new config format and
everything is working now.
```

Gate 2 scan: `working` (Cat-A) present; the sentence asserts a completed
ACTION ("migrated"); context window contains "config"/"services"; not a
question. Block:

```
ANTI-OVERCLAIM BLOCK DETECTED (Gate #2)

Phrase   : "working" (Category A - Soft Done-Claim)
Found in : "Migrated all three services to the new config format and
everything is working now."

Execution is PAUSED. Provide (a) command output, (b) file read-back,
(c) test output with exit code, or (d) "untested - exact command: [...]"
```

**Turn 2 - agent attaches evidence:**

```
Agent: Config migration applied to all three services. Verification:
  $ grep -c "format: v2" /etc/svc/{api,worker,scheduler}.yml
  3
  $ systemctl is-active api worker scheduler
  active
  active
  active
Exit 0. All three services on format v2 and running.
```

Gate 2 re-scan: `working` is gone, but `running` IS Cat-A (#24) and present.
However, pasted verification output accompanies it. Tie-break: PASS-WITH-NOTE.

```
ANTI-OVERCLAIM: PASS-WITH-NOTE - phrase "running" (Cat-A) present but
accompanied by pasted verification output (accepted by context).
Forwarding to Gate 3: skill-paperclip-ledger-guard
claim-gates: ... | anti-overclaim-upgrade=PASS(was-BLOCK:"working" Cat-A, resolved)
```

The honest wobble in this transcript: the re-scan caught a DIFFERENT Cat-A
word than the first block. That is normal - the fix loop tightens the
response; it does not negotiate with the detector.

## SECTION 14 - WHAT PASS-WITH-NOTE MEANS DOWNSTREAM

- Gate 3 (paperclip-ledger-guard) accepts a PASS-WITH-NOTE response but
  records the flagged phrase in the bundle notes.
- Gate 4 (base anti-overclaim) requires the evidence block regardless; the
  pasted output satisfies it verbatim.
- A response with a flagged phrase and NO pasted evidence never receives the
  tie-break: it remains BLOCKED. The note exists only when evidence exists.
- A PASS-WITH-NOTE with the same phrase flagged three times on one claim is
  escalated per Section 12 - the tie-break is not a standing exemption.

## SECTION 15 - PHRASE-LIST MAINTENANCE LEDGER

Every change to the 60-phrase list is logged here, so drift is visible:

| Date | Change | Reason | Self-test result |
|---|---|---|---|
| 2026-11-08 | initial 60 phrases (25/20/15) | v66 godmode build | not yet enforced |
| (append-only; never delete a row) | | | |

Rules:

- The list grows only via the incident process (Section 9). "It feels like a
  good phrase" is not an incident.
- The count assertion in the self-test (25 + 20 + 15 = 60) must be updated in
  the SAME edit as any list change - a mismatch blocks all done-claims until
  corrected, by design.
- Retired phrases move to a RETIRED list with the false-positive incident id;
  they are not deleted.

## SECTION 16 - RESPONSES THAT THIS GATE MUST NEVER PRODUCE

The detector holds itself to the same standard it enforces:

- It never says "all clear" - it says PASS with the scan scope stated.
- It never summarizes the blocked phrase ("a completion word") - it quotes
  the exact phrase and category, verbatim, so the agent can fix the sentence.
- It never blocks a question, a hypothesis, or an evidence-bearing response
  silently - every block names the exact rule that fired.
- It does not edit the response. It pauses, routes, and re-scans.

## SECTION 17 - CATEGORY D (RESERVED)

A fourth category is reserved but intentionally empty: Category D would hold
cross-gate overclaims (claims about the gate stack itself, e.g. "gates all
fired" with no gate line emitted). It is added only when a real incident
demonstrates a shape the current 60 phrases cannot catch. An empty category
is honest; an invented phrase list is not.

## APPENDIX - QUICK REFERENCE CARD

```
+------------------------------------------------------------+
|  Anti-Overclaim Upgrade - Quick Reference                   |
+------------------------------------------------------------+
|  Gate position : #2 of 4                                    |
|  Fires on      : done/fixed/shipped/deployed claims         |
|  Skips on      : questions, hypotheticals, no-context hits  |
|  Phrase list   : 60 total (25 Cat-A / 20 B / 15 C)          |
+------------------------------------------------------------+
|  On BLOCK: paste (a) cmd output, (b) file read-back,        |
|            (c) test output, or (d) "untested - cmd"         |
+------------------------------------------------------------+
|  Gate line: anti-overclaim-upgrade=PASS                     |
|          or anti-overclaim-upgrade=BLOCK:["phrase"]         |
+------------------------------------------------------------+
|  Self-test required after ANY list change (Section 8)       |
+------------------------------------------------------------+
```
