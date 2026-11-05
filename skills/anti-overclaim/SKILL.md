---
name: anti-overclaim
description: Prevents agents from marking work "done" or "shipped" without attaching verifiable evidence. Triggers before any "done" claim, before writing DONE to any status file or summary, and before closing any task in a multi-agent run. Defines blocked phrases, required proof format, and the evidence gate every completion must pass.
---

# Anti-Overclaim

## Why This Exists

The single most common failure mode across all agent runs is the false "done":

> "The migration is complete."  
> "All tests pass."  
> "The file has been written."  
> "Deployed successfully."

Each of these was stated confidently in a session where the thing had **not** happened.
The agent reasoned toward a plausible completion, found no obvious error, and reported success.
The human trusted it. The bug shipped.

This skill enforces a hard gate: **no completion claim without attached evidence that could
have come out the other way.**

---

## Trigger Conditions

Fire this gate before writing or saying any of the following in any form:

- "done", "DONE", "complete", "completed", "finished"
- "shipped", "deployed", "live", "pushed"
- "all tests pass", "green", "passing"
- "fixed", "resolved", "closed"
- "verified", "confirmed", "validated"
- Any status field set to `done` / `DONE` / `complete` in a JSON or Markdown table
- Any PR/commit description claiming the work is finished

---

## The Evidence Requirement

A completion claim is only valid if it includes **at least one** of the following:

### E1 — Command output (strongest)
Paste the actual terminal output, including the command that produced it.
The output must be from **this session**, not a previous run.

```
REQUIRED FORMAT:
$ <exact command run>
<actual stdout/stderr>
Exit code: 0   ← must be shown explicitly for critical gates
```

### E2 — File existence proof
```
$ Get-Item "path\to\file" | Select-Object FullName, Length, LastWriteTime
# or
$ ls -la path/to/file
```
The file must exist **and** have a non-zero size **and** a LastWriteTime from this session.

### E3 — Hash / checksum match
When the task requires a specific artifact (a binary, a ZIP, a database snapshot):
```
$ Get-FileHash "path\to\artifact" -Algorithm SHA256
# Expected: <hash stated in the task>
# Actual:   <hash from this command>
```

### E4 — Test suite output
```
$ dotnet test / pytest / go test ./... / npm test
# Must show: X passed, 0 failed, 0 skipped (or the exact expected counts)
# Must NOT show: "build succeeded" without test execution
```

### E5 — Remote state read-back
For deploys, pushes, API calls — read the remote state back after the action:
```
$ git ls-remote origin HEAD        # confirms push landed
$ curl -s https://example.com/health | jq .version   # confirms deploy is live
$ aws s3 ls s3://bucket/key        # confirms upload exists
```

---

## Blocked Phrases (never emit without evidence attached)

These exact phrases — and their common variants — are **banned** as standalone sentences:

| Banned phrase            | Why it fails                                      |
|--------------------------|---------------------------------------------------|
| "The task is complete."  | No evidence attached                              |
| "Everything looks good." | "Looks" = visual inspection = not a check        |
| "It should work now."    | "Should" = reasoning, not measurement            |
| "I've verified this."    | Verification requires a command; name it         |
| "Tests are passing."     | Without pasted output, this is a guess           |
| "Done ✓"                 | The checkmark adds false confidence              |
| "Successfully deployed." | Without a read-back, this is an assumption       |

**Safe alternative:** replace any blocked phrase with the evidence block that would prove it,
or with an honest "untested — run `<exact command>` to verify."

---

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

If the verdict is UNTESTED, the task is NOT done — it is pending verification.
Do not close the task. Do not write DONE to any status field.

---

## Self-Check Before Any Completion Claim

Answer these four questions before writing "done":

1. **Can I name the command that proves this?**
   If no → the claim is a guess. Do not make it.

2. **Did I run that command in THIS session?**
   If no → run it now. Paste the output.

3. **Could that command have returned a non-zero exit code or failure output?**
   If it could not fail → it is not a real check. Find one that can.

4. **Is the output from the current state, not a previous run?**
   If stale → re-run it. Timestamps matter.

All four must be "yes" before the claim is valid.

---

## Multi-Agent Context

In a fleet run, each seat must:

- Attach evidence to its own output JSON under `"evidence"` key
- Never inherit "done" status from another seat without re-verifying locally
- If a dependency seat reports done but its evidence is missing, treat it as UNTESTED

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

A status of `"done"` without an `"evidence"` block is invalid and must be treated as
`"unverified"` by the orchestrator.

---

## Integration with /honestscore

This skill gates **entry** to /honestscore. You cannot meaningfully score a deliverable
that has not been proven to exist. The flow is:

1. Run Anti-Overclaim gate → attach evidence
2. Only if evidence passes → invoke /honestscore
3. Iterate on /honestscore defects until score ≥ threshold
4. Then and only then write the final DONE status

Skipping step 1 and going straight to /honestscore is itself an overclaim.
