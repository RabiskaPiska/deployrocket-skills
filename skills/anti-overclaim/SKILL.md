---
name: anti-overclaim
description: Prevents agents from marking work "done" or "shipped" without attaching verifiable evidence. Triggers before any "done" claim, before writing DONE to any status file or summary, and before closing any task in a multi-agent run. Defines blocked phrases, required proof format, and the evidence gate every completion must pass.
alwaysFire: true
---

# Anti-Overclaim

## SUPERSESSION AND ALIAS MAP

This skill is the BASE LAYER (Gate 4) of the four-gate claim-verification
chain. It pairs with its upstream twin:

```
Gate 1: skill-ground-truth-harness       external evidence must be pasted
Gate 2: skill-anti-overclaim-upgrade     live 60-phrase scanner (LEAD)
Gate 3: skill-paperclip-ledger-guard     HMAC bundle sign-off
Gate 4: anti-overclaim (THIS SKILL)      base evidence gate - final check
```

- **The lead** for phrase detection and gate ordering is
  `skill-anti-overclaim-upgrade` - it owns the live 60-phrase list, the
  detection algorithm, and the self-test. New phrases are added THERE.
- **This skill remains active** as the base layer: it defines the evidence
  requirement (E1-E5), the completion-evidence block format, and the
  multi-agent evidence schema. Its phrase table below is reference material
  kept in sync with the lead, not the live detector.
- **The alias map fires either.** If only one of the two ids is invoked, the
  chain still runs: this skill is the last gate before a done-claim is
  accepted, and it never approves a claim that lacks the evidence block,
  regardless of what earlier gates passed.

Division of labor, one line each:

| Concern | Owner |
|---|---|
| Live blocked-phrase list + detector | skill-anti-overclaim-upgrade |
| Evidence-type definitions + proof format | THIS SKILL (authoritative) |
| Cryptographic bundle sign-off | skill-paperclip-ledger-guard |
| Quality scoring | honestscore (after this gate) |

## Why This Exists

The single most common failure mode across all agent runs is the false "done":

> "The migration is complete."
> "All tests pass."
> "The file has been written."
> "Deployed successfully."

Each of these was stated confidently in a session where the thing had **not**
happened. The agent reasoned toward a plausible completion, found no obvious
error, and reported success. The human trusted it. The bug shipped.

This skill enforces a hard gate: **no completion claim without attached
evidence that could have come out the other way.**

## Trigger Conditions

Fire this gate before writing or saying any of the following in any form:

- "done", "DONE", "complete", "completed", "finished"
- "shipped", "deployed", "live", "pushed"
- "all tests pass", "green", "passing"
- "fixed", "resolved", "closed"
- "verified", "confirmed", "validated"
- Any status field set to `done` / `DONE` / `complete` in a JSON or Markdown table
- Any PR/commit description claiming the work is finished
- Any handoff message declaring a task complete before handoff

The gate fires BEFORE the claim is written - not after. Rewriting a done
message to be softer does not satisfy the gate; attaching evidence does.

## The Evidence Requirement

A completion claim is only valid if it includes **at least one** of the
following. The types are ordered by strength; when in doubt, use the
strongest available.

### E1 - Command output (strongest)

Paste the actual terminal output, including the command that produced it.
The output must be from **this session**, not a previous run.

```
REQUIRED FORMAT:
$ <exact command run>
<actual stdout/stderr>
Exit code: 0   <- must be shown explicitly for critical gates
```

Rules:

- The command is stated exactly as typed, including flags.
- Output is pasted verbatim - summarized output is testimony.
- Exit code shown unpiped. In PowerShell check `$LASTEXITCODE`; in bash
  `echo $?` immediately after the command.
- Evidence older than the last edit is void - re-run after the final change.

### E2 - File existence proof

```
$ Get-Item "path\to\file" | Select-Object FullName, Length, LastWriteTime
# or
$ ls -la path/to/file
```

The file must exist **and** have a non-zero size **and** a LastWriteTime
from this session. A zero-byte file is not an artifact; a stale timestamp
is not evidence of this session's work.

### E3 - Hash / checksum match

When the task requires a specific artifact (a binary, a ZIP, a snapshot):

```
$ Get-FileHash "path\to\artifact" -Algorithm SHA256
# Expected: <hash stated in the task>
# Actual:   <hash from this command>
```

The hash proves content, not just existence - and it must match the hash
stated in the task, recomputed in this session. A hash quoted from memory
is a guess wearing a costume.

### E4 - Test suite output

```
$ dotnet test / pytest / go test ./... / npm test
# Must show: X passed, 0 failed, 0 skipped (or the exact expected counts)
# Must NOT show: "build succeeded" without test execution
```

Rules:

- The runner output is pasted, including the summary line and exit code.
- Skipped tests are named; a skip of the test that encodes the requirement
  voids the claim.
- For concurrency-touched code: the race-mode run (`go test -race` or an
  equivalent stress loop) is the valid evidence; a serial pass is not.

### E5 - Remote state read-back

For deploys, pushes, API calls - read the remote state back after the action:

```
$ git ls-remote origin HEAD        # confirms push landed
$ curl -s https://example.com/health | jq .version   # confirms deploy is live
$ aws s3 ls s3://bucket/key        # confirms upload exists
```

Rules:

- Read back through the PUBLIC path the user actually hits.
- The read-back must show a distinguishing value from THIS change (new
  field, new version, new count) - "it responds" is not evidence of the
  new version being served.

## Blocked Phrases (never emit without evidence attached)

These exact phrases - and their common variants - are **banned** as
standalone sentences:

| Banned phrase            | Why it fails                                      |
|--------------------------|---------------------------------------------------|
| "The task is complete."  | No evidence attached                              |
| "Everything looks good." | "Looks" = visual inspection = not a check        |
| "It should work now."    | "Should" = reasoning, not measurement            |
| "I've verified this."    | Verification requires a command; name it         |
| "Tests are passing."     | Without pasted output, this is a guess           |
| "Done [checkmark]"       | The checkmark adds false confidence              |
| "Successfully deployed." | Without a read-back, this is an assumption       |
| "All good now."          | No check named, nothing pasted                   |
| "Should be fine."        | Confidence is not measurement                    |
| "I'm confident this works." | Confidence is not evidence                     |

**Safe alternative:** replace any blocked phrase with the evidence block
that would prove it, or with an honest "untested - run `<exact command>` to
verify."

## Required Proof Format

Every task completion entry must follow this template:

```
## Completion Evidence

**Claim:** <what was done in one sentence>

**Evidence type:** E1 / E2 / E3 / E4 / E5  (pick one from the list above)

**Command run:**
$ <command>

**Output:**
<paste real output here>

**Exit code:** <0 or non-zero + explanation>

**Verdict:** PROVEN DONE / UNTESTED (run `<command>` to verify)
```

If the verdict is UNTESTED, the task is NOT done - it is pending
verification. Do not close the task. Do not write DONE to any status field.

### The four questions before any completion claim

1. **Can I name the command that proves this?**
   If no - the claim is a guess. Do not make it.

2. **Did I run that command in THIS session?**
   If no - run it now. Paste the output.

3. **Could that command have returned a non-zero exit code or failure output?**
   If it could not fail - it is not a real check. Find one that can.

4. **Is the output from the current state, not a previous run?**
   If stale - re-run it. Timestamps matter.

All four must be "yes" before the claim is valid.

## Multi-Agent Context

In a fleet run, each seat must:

- Attach evidence to its own output JSON under the `"evidence"` key
- Never inherit "done" status from another seat without re-verifying locally
- If a dependency seat reports done but its evidence is missing, treat it
  as UNTESTED

### Output JSON schema addition

```json
{
  "seat": 13,
  "status": "done",
  "evidence": {
    "type": "E1",
    "command": "Get-Item 'path/to/output.md' | Select Length",
    "output": "Length: 4821",
    "exit_code": 0
  }
}
```

A status of `"done"` without an `"evidence"` block is invalid and must be
treated as `"unverified"` by the orchestrator.

### Evidence-block validation rules

The orchestrator (or the reviewing gate) validates the evidence block:

| Check | Invalid if |
|---|---|
| Type present | missing or not one of E1-E5 |
| Command stated | absent, or a description instead of a command |
| Output pasted | absent, or reads like a paraphrase |
| Exit code shown | absent for critical gates |
| This session | timestamp/older output - treat as UNVERIFIED |
| Non-zero handled | exit != 0 without an explanation = FAIL |

A status of `"done"` with evidence that fails any check reads as
`"unverified"`; the seat is re-queued.

## Integration with /honestscore

This skill gates **entry** to /honestscore. You cannot meaningfully score a
deliverable that has not been proven to exist. The flow is:

1. Run Anti-Overclaim gate -> attach evidence
2. Only if evidence passes -> invoke /honestscore
3. Iterate on /honestscore defects until score >= threshold
4. Then and only then write the final DONE status

Skipping step 1 and going straight to /honestscore is itself an overclaim.

## Integration with the other gates

Full chain position (this skill is last, by design):

```
Gate 1: skill-ground-truth-harness      (external evidence must be pasted)
Gate 2: skill-anti-overclaim-upgrade    (live 60-phrase scan, PASS/BLOCK)
Gate 3: skill-paperclip-ledger-guard    (HMAC-signed bundle)
Gate 4: anti-overclaim (THIS SKILL)     (evidence block + final approval)
```

What each gate needs from this one:

- The completion-evidence block (Section "Required Proof Format"), pasted.
- The evidence-type label (E1-E5).
- An explicit verdict of PROVEN DONE or UNTESTED - nothing else.

A claim that reaches this gate without the block is rejected with:
`anti-overclaim=BLOCKED:no-evidence-block`.

## Status-Table Rules (the DONE-cell discipline)

Status tables are where overclaims hide, because a green cell reads as
verified without saying so:

- Every cell that says done/DONE/complete must reference its evidence
  (a ledger row, a bundle, or a pasted output in the same document).
- A cell that says `done` with no pointer is rewritten to `unverified`.
- `wip`, `untested`, and `blocked:<reason>` are always valid cell values;
  they are preferred over a naked `done` with no evidence.
- In multi-agent boards: a seat may never set another seat's row to done.
  Only the owning seat (or the orchestrator after re-verification) may.

## Worked Example - a full evidence block

```
## Completion Evidence

**Claim:** Migration 0003_add_index applied to production database.

**Evidence type:** E5 (remote state read-back)

**Command run:**
$ psql $PROD_URL -c "SELECT indexname FROM pg_indexes WHERE tablename='orders'"
       -c "\d orders" | grep idx_orders_created_at

**Output:**
      indexname       | tablename
----------------------+----------
 idx_orders_created   | orders
(1 row)

**Exit code:** 0

**Verdict:** PROVEN DONE
```

Counter-example (INVALID - the shape is right, the content is not):

```
**Evidence type:** E1
**Output:** [migration succeeded]      <- described, not pasted
**Verdict:** PROVEN DONE
```

A placeholder in the Output field fails the gate regardless of how
confident the prose reads.

## Self-Check Checklist (run before emitting any done-claim)

- [ ] The claim is one sentence and specific
- [ ] An evidence type (E1-E5) is named
- [ ] The command is stated exactly as typed
- [ ] The real output is pasted, not described
- [ ] The exit code is shown, unpiped
- [ ] The evidence is from THIS session (post-last-edit)
- [ ] Verdict reads PROVEN DONE or UNTESTED - never a bare "done"
- [ ] If UNTESTED: the exact verify command is provided, no DONE written

## FAQ

**Q: I have the evidence but it is long.**
Paste the relevant slice plus the exit code, and name the file holding the
full output. Truncation is fine; absence is not.

**Q: The evidence is from a teammate's run.**
Label it INHERITED and re-run the one command locally before the claim.
Inherited green is unverified until re-run locally.

**Q: The evidence exists but I cannot paste it (too large, secret-bearing).**
Paste the filtered slice that proves the claim + a hash of the full output
(`Get-FileHash` / `sha256sum`). The hash ties the pasted slice to the whole.

**Q: Does UNTESTED ever become done?**
Only by running the verify command in a later session and re-passing this
gate. "Pending verification" is a status, not a failure - a false done is
the only unrecoverable state.

## INCIDENT TAXONOMY (the shapes this gate exists to stop)

Each shape below has shipped a real false-done in fleet history. The
evidence requirement is mapped to the shape it kills:

| Shape | What happened | Gate answer |
|---|---|---|
| Reasoned-to-completion | Agent found no obvious error, reported success | E1: run the check in this session |
| The stale green | "Tests pass" quoted from a run before the last edit | E4 re-run after final edit |
| The summarized paste | Output described, not shown | Evidence must be verbatim |
| The inherited done | Seat B cites Seat A's green without re-running | Re-run locally or mark UNVERIFIED |
| The internal-path deploy | /health 200 on the internal net, dead publicly | E5 through the public path |
| The placeholder artifact | "TODO" where a hash should be | E3 rejects placeholder list |
| The silent skip | Test suite "passed" - one test was skipped | E4 with named skips |
| The self-confirming probe | pgrep matched its own command line | Verify side-effects, not process names |
| The count without source | "All 12 endpoints migrated" | C6-style: recompute from a file |
| The adverb claim | "successfully", "smoothly", "cleanly" | Not evidence - replace with the block |

New shapes enter here by incident and map to an evidence type. The taxonomy
is never pruned; retired entries are marked, not deleted.

## SEAT-TO-SEAT EVIDENCE PROPAGATION

In a multi-seat run, "done" flows downhill and lies at every hop unless
pinned. Rules:

1. A seat's done-claim is valid ONLY with its own evidence block. There is
   no such thing as second-hand done.
2. A downstream seat consuming an upstream artifact re-runs the ONE check
   that would catch the upstream defect (a hash, a count, a smoke test)
   before its own done-claim.
3. If upstream evidence is missing, the downstream row reads
   `blocked:upstream-unverified` - never `done` on faith.
4. The orchestrator's merge is itself a done-claim: the merged artifact
   gets its own E2/E3 evidence (exists, non-zero, hash stated).

### Propagation schema

```json
{
  "seat": 7,
  "status": "done",
  "evidence": { "type": "E1", "command": "...", "output": "...", "exit_code": 0 },
  "consumes": [
    { "seat": 12, "artifact": "lib/parser.out", "check": "sha256", "value": "ab12..." }
  ]
}
```

The `consumes` entries are the seat's own re-verification of upstream
artifacts. A missing `consumes` entry for a consumed artifact is an
overclaim by omission.

## DEPLOY-SPECIFIC GATE

Deploys get the strictest reading of E5 - every deploy claim carries:

1. The public-path read-back (the URL a user hits), showing a value that
   distinguishes THIS deploy (version string, new copy, new field).
2. The rollback target named BEFORE the deploy: previous tag/artifact, and
   the one-command rollback path.
3. A post-deploy error-rate look (or an explicit statement that traffic is
   too low to observe one yet, with the check scheduled).

A deploy claimed "done" with an internal-health-only check is the classic
false positive: internal green, public path broken.

## EVIDENCE IN LONG-RUN STATE FILES

When the run persists state to disk (goal dirs, runlogs, status files):

- Status fields in state files follow the same rule: `done` requires an
  evidence pointer in the adjacent field, or it reads `unverified`.
- The runlog's "DONE" section lists each completed item with its evidence
  command, so a fresh session can re-verify without archaeology.
- CONTINUE-HERE files state UNFINISHED work as UNTESTED with the exact
  command to resume verification - never a bare "mostly done".

## ANTI-PATTERNS (each shipped a real false done)

1. The bare "done." - no evidence, no verdict block.
2. The rephrase - "wrapped up", "in place", "all set" used to dodge the
   phrase list while still claiming completion without evidence. The
   evidence requirement is the gate; phrasing is not the point.
3. The stale verdict - PROVEN DONE carried across edits without re-running.
4. The optimistic table - a status row set to done "so the report looks
   complete". The orchestrator re-queues any row without evidence.
5. The borrowed green - quoting another seat's output as your own without
   the INHERITED label.
6. The skipped announcement - declaring completion of the verification step
   itself without running it.

## FAQ (continued)

**Q: I ran the command in a previous session.**
Then re-run it. Evidence is session-scoped; "it passed before" describes a
previous world.

**Q: The evidence is a screenshot.**
Screenshots are secondary evidence at best: full-scroll, real display size,
and paired with a machine-readable check. A picture of a terminal is not an
exit code.

**Q: The evidence exists but the claim is a forecast ("will be done by X").**
The gate does not block forecasts - only assertions of completed fact. A
forecast that dresses itself as a result ("the deploy will be live shortly,
done for today") is a Category A overclaim in the live list.

**Q: Can I mark a task done with UNTESTED evidence?**
No. The task is PENDING VERIFICATION. Close the task only on PROVEN DONE.
Writing DONE with an UNTESTED verdict is the exact violation this gate
exists to stop.

## Quick Reference Card

```
+------------------------------------------------------------+
|  anti-overclaim (base layer, Gate #4)                       |
+------------------------------------------------------------+
|  No completion claim without attached evidence.             |
|  Evidence types: E1 cmd output | E2 file proof |            |
|                  E3 hash match  | E4 test output |          |
|                  E5 remote read-back                        |
+------------------------------------------------------------+
|  Required block: Claim / Type / Command / Output /          |
|                  Exit code / Verdict                        |
+------------------------------------------------------------+
|  Verdict: PROVEN DONE or UNTESTED (with verify command)     |
|  Anything else = overclaim                                  |
+------------------------------------------------------------+
|  Status tables: every done cell carries an evidence pointer |
+------------------------------------------------------------+
```
