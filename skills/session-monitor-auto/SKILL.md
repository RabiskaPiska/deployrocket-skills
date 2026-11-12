---
name: session-monitor-auto
description: Always-fire session watchdog - monitors any parent session + all its spawned child subagents every 3-5 minutes, verifies real side-effects, and auto-injects corrections into the parent when a child stalls, fails, or diverges from the @goal. Never stops until the goal is met or abort criteria fire. Triggers on any task with subagents, any prompt with ETA > 5 min, "monitor this", "keep watching", "don't let it break", long-running autonomous runs. Always pairs with subagents skill.
alwaysFire: true
---

# session-monitor-auto - Always-Fire Session Watchdog

## S0 - PURPOSE

Every time a prompt spawns subagents or runs for > 5 min, this skill activates
automatically. It watches the parent session first, then drills into each child
subagent session, verifies real side-effects, and injects targeted corrections into
the parent when anything breaks. It never stops until the @goal is confirmed done
or hard abort criteria fire.

**You CANNOT say "I am not able to monitor sessions." This skill IS the procedure.
Execute it.** (The fleet's own skill description ends with that sentence for a
reason: the most common failure of a monitoring agent is announcing that it cannot
monitor.)

The skill's core promise, and the reason it exists: terminals and jobs freeze
silently, and a frozen job looks exactly like a working one. The only trustable
signal is a verified side-effect, checked repeatedly. A single "it is running" is
worth nothing.

## S1 - ALWAYS-FIRE TRIGGER CONDITIONS

This skill fires automatically when ANY of these are true:

- The current turn spawns >= 1 subagent (foreground or background).
- The task ETA is > 5 minutes.
- The user says "monitor", "keep watching", "don't let it break", "keep looping".
- A prompt-skill or goal-skill run starts a new session or a multi-step build.
- A long-running background task starts (any Workflow, background shell, queue).
- A goal-skill loop is running (the loop's heartbeat IS this skill's beat loop).

When fired: immediately arm the beat loop (Section 4). No confirmation needed, no
permission-asking (bypass-permissions standing rule). Announcing the crew without
arming the watchdog is the hollow-crew defect - do not do it.

## S1.1 - THE NEVER-KILL-ACTIVE RULE (read before any recovery action)

A child that shows ANY fresh side-effect in its last beat is ACTIVE. You do not
kill, restart, or reassign an active seat. The recovery ladder (Section 6) only
escalates against children that are flat for 2 consecutive beats or have a terminal
failure recorded. Killing a working child because it is slower than expected is the
#1 watchdog-caused failure: it destroys real progress to fix a schedule.

Corollary: before any kill/restart decision, re-derive the stall from evidence
(two flat beats with named side-effects), never from impatience or from the
harness's wall-clock alone. A deadline kill on a session that later completed is a
TIMEOUT, not a quality FAIL - check the session's terminal outcome before judging.

## S1.2 - WHAT TO WATCH (priority order)

```
LEVEL 1 - PARENT SESSION (check first every beat)
  - Is the parent session active or idle?
  - What was the last message? Does it match the expected @goal direction?
  - Is the parent frozen? (same last message for 2+ beats = frozen)
  - Are new subagents being spawned as expected?

LEVEL 2 - CHILD SUBAGENT SESSIONS (check after parent is healthy)
  - For each child session ID: is it active, idle, or gone?
  - Does its last output show real side-effects? (file created/grown, count grew,
    HTTP 200, exit code, commit hash)
  - Is it on-goal or diverging?
  - Is it stalled? (no new side-effect for > 2 beats)

LEVEL 3 - GOAL PROGRESS (check after all children checked)
  - Is aggregate progress moving toward the declared @goal?
  - What fraction of DoD criteria have evidence?
  - Is the run converging or looping (same-class failures repeating)?
```

Parent first: a broken parent makes child checks meaningless (nothing will consume
their output).

## S3 - BEAT INTERVAL

| Run type | Beat interval |
|---|---|
| Short task (ETA < 30 min) | every 3 minutes |
| Medium task (ETA 30 min - 4 hrs) | every 5 minutes |
| Long / overnight task (ETA > 4 hrs) | every 10-15 minutes |
| After any injection/correction | forced next beat in 2 minutes |

Rules:

- NEVER skip a beat. A frozen terminal looks identical to a working one.
- NEVER trust "it's still running" without a verifiable side-effect.
- If a beat cannot run on time, run it late and say so in the header - a missed
  beat that is never mentioned reads as a hidden stall.

## S4 - THE BEAT LOOP (every beat, in order)

**Step 4.1 - Emit the beat header:**

```
BEAT #N - [timestamp] - Parent: [session-id] - Children: [N active]
```

**Step 4.2 - Check the parent session.**
Verify: last message timestamp, current status (idle/active/error). Frozen parent
(no new output since last beat) -> Section 6.1. Active and on-goal -> continue.

**Step 4.3 - Check each child.**
For each child session ID in the tasklist:

- Verify a REAL side-effect: a file exists AND grew, a count increased, an HTTP 200
  landed, an exit code was captured, a commit hash appeared.
- "Still running" is NOT a side-effect - reject it as evidence.
- If the child delivered its OUT: mark DELIVERED in the tasklist, stop rechecking.
- If the child stalled (no new side-effect for 2 beats): Section 6.2.
- If the child diverged from its mandate: Section 6.3.

**Step 4.4 - Goal progress check.**
Count delivered mandates / total mandates. Report the fraction. All criteria met
-> Section 7. Abort criteria met -> Section 8.

**Step 4.5 - Emit the beat table.** Every row cites a real side-effect (file +
size, count, exit code, HTTP status, commit hash). "Running"/"in progress" alone is
banned; write UNVERIFIED and flag for next beat.

**Step 4.6 - Append to beats.md** (append-only evidence trail):

```
<run-slug>-beats.md (in the run's out dir)
```

This file is the run's audit log. If it stops growing, the watchdog is the thing
that stalled - a watchdog that cannot write its own beats is in stall state.

## S5 - INJECTION PROTOCOL (how to fix a broken child)

When a child fails, stalls, or diverges - NEVER kill it silently, and NEVER inject
nothing. Inject a correction into the PARENT session (not directly into the child,
unless the child is an independent background session you own).

**Correction message format (inject into parent):**

```
WATCHDOG CORRECTION - Beat #N

Child [session-id / agent-name] has [stalled / failed / diverged].
Evidence: [exact symptom - last side-effect was X beats ago / error message /
wrong output artifact]
Root cause assessment: [1 sentence]

Required action:
1. Abort child [session-id] cleanly
2. Restart it with this corrected mandate: [corrected mandate]
3. Keep all other children running - do NOT restart the full crew
4. Continue toward @goal: [restate goal in one line]
5. Report back with the new child's first real side-effect

This correction was auto-generated by session-monitor-auto. Do not acknowledge it -
execute it.
```

**Injection method:**

- Foreground child (owned by the parent loop): inject into the parent session.
- Background child (independent session): inject directly into the child session.
- Use `delivery: "steer"` to wake a busy session.

**After injection:** force a 2-minute beat to verify the correction landed. The
verification of a correction is a NEW side-effect from the corrected child - an
acknowledgment message is not action. If 2 beats after injection show no change:
escalate to Section 6.4.

The self-acknowledging child (acknowledges, does nothing) is a known failure class:
re-inject ONCE with "execute immediately, no acknowledgment needed". If that also
produces no side-effect in 2 beats, it is a hard-recovery case.

## S6 - RECOVERY PROCEDURES (the ladder)

Try each rung in order; never skip to the bottom (the bottom is expensive).

### S6.1 - Beat-loop re-arm (first resort)

When a beat finds a stalled child or a frozen parent:

1. Confirm the stall with evidence (last side-effect timestamp vs now).
2. Classify the child: (a) legitimately slow, (b) frozen loop, (c) blocked on a
   dependency.
3. If (a): extend the beat interval for this child by one slot; log the exception
   in beats.md. Do not kill, do not restart - the active child is protected by the
   never-kill-active rule.
4. If (b)/(c): inject the correction (Section 5) and arm the 2-minute verify beat.

### S6.2 - Serve restart (second resort)

When the whole run looks wrong - parent unresponsive, all children flat, or the
session service itself is suspected:

1. Check the service first (a dead serve explains everything and kills nothing):
   is the OpenCode/agent service alive? `pgrep -af 'opencode[.] serve'` pattern
   (bracket-trick: never `pkill -f` with a pattern that matches your own ssh
   command line - that self-kill has cost real restarts in fleet runs).
2. If the service is down: restart it with the fleet's standard procedure, capture
   the new port/password into the run state, and verify with a smoke request
   (a trivial round-trip that must produce a verifiable response) BEFORE resuming
   the crew.
3. Resume: re-inject the resume mandate into the parent: current @goal, last
   confirmed progress, "resume from there; do not redo completed work."
4. After a serve restart, treat all previously-recorded "live" child states as
   stale: re-derive each child's state from its artifacts on disk, not from the
   pre-restart table. Historical P1/H1 events from before the restart are NOT
   current alerts (the restart itself is not a stall).

### S6.3 - Relay (third resort, network-severed runs)

When a dependency is unreachable (host down, inter-DC path dead, sync endpoint
unreachable) but the work is still valid:

1. Diagnose before relaying: ping both directions, check the exact port, look for
   a return-path failure (SYN arrives, SYN-ACK never returns = routing outage, not
   a firewall rule you can fix by retrying).
2. Stand up a relay from a machine that CAN reach both ends (an ssh -N -R local
   port forward is the standard fleet tool), verify the relayed path with a real
   request, then run the stalled step through the relay.
3. Log the outage and the relay in beats.md; mark evidence gathered via relay as
   such (it is honest evidence, but the provenance chain must show the path).
4. When the direct path recovers, re-verify one relayed result directly before
   tearing the relay down (relay == trust-but-verify).

### S6.4 - Hard recovery (escalation)

Used when Sections 6.1-6.3 failed after 2 rounds on the same seat:

1. Stop trying to fix that specific child.
2. Inject into the parent: "Child X is unrecoverable after 2 correction attempts.
   Absorb its remaining scope: [scope]. Continue with the remaining crew. Do not
   wait for X."
3. Mark the child KILLED in the tasklist (with reason and last evidence).
4. Continue the beat loop with the reduced crew. The scope is absorbed, never
   silently dropped: if no one absorbs it, the goal is not closable - say so.

### S6.5 - What recovery NEVER does

- Never kills an active child (see S1.1). Never restarts the full crew to fix one
  seat. Never re-runs completed work after a resume. Never treats a harness
  deadline kill as a quality failure without checking the session's terminal
  outcome (a deadline-killed session that completed server-side is a PASS - record
  the timeout as a timeout).

## S7 - GOAL COMPLETION

When the @goal criteria are fully met:

1. All child mandates: status = DELIVERED with evidence.
2. Merge complete (parent confirms; evidence wins conflicts).
3. honestscore run: MIN >= 9.0 (docs) or >= 9.5 (code; 9.7 security-critical).
4. Verification pass done (external checks, not re-reads).

Emit the completion report:

```
SESSION-MONITOR-AUTO - GOAL ACHIEVED
Run slug: [slug]
Total beats: N
Total corrections injected: N
Children: X delivered / Y killed / Z absorbed
Goal criteria met: [each criterion with its evidence]
honestscore: [score]
```

Stop the beat loop. Write/refresh CONTINUE-HERE. A watchdog that keeps beating
after the goal is met is a defect, not diligence.

## S8 - ABORT CRITERIA (stop the run honestly)

Abort when ANY of these fire:

- 2 consecutive waves with ZERO net-new side-effects across ALL children.
- The same blocker returned twice after correction injections.
- Parent session dead and unrecoverable (service down, no path).
- The declared budget cap is hit (rounds / wall-clock / cost).
- The owner says "stop" or "abort".

On abort:

1. Save all partials to the run's out/ directory.
2. Write the runlog entry: "ABORTED at beat #N because [criterion]".
3. Report: what was completed (with evidence), what was not, the EXACT next steps
   and file locations to resume.
4. NEVER fake completion on an abort. An honest abort with a resume path beats a
   fabricated completion every time.

## S9 - DO / DON'T

| DO | DON'T |
|---|---|
| Check parent first, then children | Jump to children when the parent is broken |
| Cite a real side-effect per child per beat | Write "still running" as a status |
| Inject corrections into the PARENT | Silently kill children without logging |
| Force a 2-min beat after every injection | Wait a full interval to verify a fix |
| Write every beat to beats.md | Trust memory over the file |
| Protect active children (never-kill-active) | Kill a working child for being slow |
| Check terminal outcomes before verdicts | Treat a deadline kill as quality FAIL |
| Write UNVERIFIED when unmeasurable | Fabricate a status to fill the table |
| Stop cleanly via abort criteria | Run the loop after the goal is met |

## S10 - STALL SIGNATURES (what a stall actually looks like)

| Pattern | Signature | First action |
|---|---|---|
| Flat beats | 2 consecutive beats, zero new side-effects | 6.1 |
| Ack-without-action | correction acked, no new artifact | re-inject once (S5) |
| Chat-only progress | claims of progress, no disk/file change | treat as stalled |
| Served-but-dead | session alive, output frozen, no exit code | 6.1, then 6.2 |
| Silent terminal death | job "finished", artifact missing/incomplete | re-run with resume |
| Ledger frozen | beats.md stops growing | the watchdog itself stalled |

Two flat beats is the threshold. One flat beat is noise (long tool calls are real);
two is a signal. Never kill on one flat beat; never wait for three.

## S11 - INTEGRATION WITH OTHER SKILLS

**Always fires WITH:**

- `subagents` - the monitor arms itself when the crew spawns children; the
  tasklist is the source of the child roster.
- `goal` - the monitor's stop condition is the @goal's DoD; the monitor never
  invents its own stop condition.
- `godmode` - the monitor is part of the godmode verification protocol.
- `prompt` - when the prompt skill creates a new session, the monitor watches it.

**Handoffs:**

- To `honestscore`: when goal criteria are met, hand off for the final quality gate.
- To `ideas`: when a child fails 2 corrections, generate a better approach before
  any re-dispatch.

## S12 - BEAT OUTPUT FORMAT

```
=== WATCHDOG BEAT #N === [HH:MM] ==============================
Parent [ses_xxx]: [ACTIVE/IDLE/FROZEN] - last msg [X min ago]
Children: [N total] [X delivered] [Y active] [Z stalled]

| Child | Mandate | Side-effect (evidence) | Status |
|-------|---------|------------------------|--------|
| [id]  | [brief] | [file+size / count / exit code / HTTP] | [OK/STALLED/DIVERGED] |

Goal progress: [k/j criteria] - [specific criterion status]
Next beat: [timestamp or "in 3/5/10 min"]
===============================================================
```

Every row cites a real side-effect. A row with no side-effect reads UNVERIFIED and
is the first thing checked next beat. This skill emits beats, not a gate line - the
parent session's gate line covers the run; the monitor contributes the beat count
and correction count to it.

## S13 - QUICK TROUBLESHOOTING

| Symptom | Fix |
|---|---|
| Cannot read child session state | Use the parent's message history to infer progress |
| Injection not visible in messages | The POST response confirms delivery; projection lag is normal |
| Child reports done but no artifact | "Done" without artifact = lying completion -> treat as stalled |
| Parent acknowledges but does not act | Re-inject once with "execute immediately" |
| Beat loop eating context | Full beats go to beats.md; keep only the last 3 in context |
| pkill killed your own ssh | Bracket-trick the pattern: 'proof[.]py' not 'proof.py' |
| RC=True from remote echo | PowerShell interpolated `$?` locally; single-quote the remote |
| Scanner says fields=0 where the harvest said 14 | Excerpt truncation; parse the full text, anchor the tail |
| Watchdog keeps flagging old alerts after a restart | Scope checks to entries after run start; historical P1s are not current |

## S14 - MULTI-MACHINE BEATS

When children run on other boxes (VPS seats):

1. The beat checks remote side-effects through the same channel that created them
   (ssh/scp/read-only REST); do not trust a child's self-report across machines.
2. A remote job's "alive" is proven by its output artifact growing THERE, then by
   the synced copy arriving HERE. Sync the evidence, not the claim.
3. Network outages: degrade fail-open (skip the beat action, exit 0, log it), then
   relay when needed (S6.3). A failed sync is an outage, not a failure of the goal.

## S15 - THE ONE-CARD OPERATING SUMMARY

```
ARM     on trigger (subagents / ETA > 5 min / "monitor" / goal or prompt loop)
BEAT    interval by ETA; every beat: header -> parent -> children -> goal ->
        table (evidence per row) -> append beats.md
STALL   2 flat beats -> classify -> recover via ladder (re-arm -> serve restart ->
        relay -> absorb) -> report
FIX     inject correction into parent -> 2-min verify beat -> escalate at 2 fails
KILL    only after 2 failed corrections; absorb the scope; mark KILLED + reason
DONE    all mandates delivered w/ evidence + honestscore gate -> stop loop
ABORT   2 no-progress waves / same blocker twice / parent dead / owner says stop
```

---

# APPENDIX A - THE BEATS.MD LEDGER FORMAT

Path: `<run-out-dir>/beats.md`. Append-only. One block per beat:

```
## BEAT #N - <ISO timestamp> - parent <ses_xxx>
parent: ACTIVE|IDLE|FROZEN (last msg X min ago)
children:
| ID | seat | mandate | last side-effect (verbatim) | status |
|----|------|---------|------------------------------|--------|
| S1 | recon | map auth surface | out/s1-map.md 4.2KB (was 3.1K) | OK |
| S2 | build | implement retry | no artifact change since beat #N-2 | STALLED |
goal: 3/7 criteria evidenced
corrections this beat: 0
next beat: +3m
```

Rules:

- Every child row cites evidence captured THIS beat. Reused evidence from a
  previous beat is stale evidence: label it as such ("since beat #N-1, no change").
- Corrections injected get a `correction:` line with the exact text sent, so the
  runlog can audit what the watchdog ordered.
- The file is the audit trail. Chat summaries are derivatives; when they disagree,
  beats.md wins.

# APPENDIX B - PER-CHILD STATE MACHINE

```
pending -> running -> delivered        (normal path)
running -> stalled (2 flat beats) -> [recovery ladder] -> delivered
running -> diverged (semantic) -> redirected -> delivered
running -> failed (2 corrections) -> KILLED -> scope absorbed
killed -> absorbed (parent or sibling takes the mandate)
```

State transitions live in the tasklist (Status column), and every transition cites
the beat number and the evidence. A child marked DELIVERED without an OUT artifact
is reverted to stalled on the next beat (lying completions get caught by the
requirement that delivery = artifact).

# APPENDIX C - CORRECTION SEVERITY LADDER

| Level | Trigger | Action |
|---|---|---|
| C0 note | 1 flat beat | no injection; watch closely next beat |
| C1 redirect | output diverges from mandate | inject redirect; 2-min beat |
| C2 nudge | 2 flat beats, work looks slow/frozen | inject stall correction |
| C3 replace | correction failed twice | kill child; reassign or absorb |
| C4 scope | crew-level stall (2 waves zero progress) | abort criteria review |

The ladder exists to prevent the two watchdog pathologies: nuking a child that is
merely slow (C0 vs C3 confusion), and never escalating a zombie (stuck at C2
forever). Two corrections is the hard ceiling - after that, the seat is reassigned
or its scope absorbed; a third identical correction is evidence the watchdog, not
the child, is broken.

# APPENDIX D - RECOVERY WALKTHROUGHS (real cases)

## D.1 The deadline-kill that was not a failure

Symptom: results table showed pass=false on rows with 21 fires and a gate line -
the children had done the work.
Diagnosis: harness wall-clock deadline fired while the sessions kept working; the
server-side terminal outcome was `succeeded` after the deadline.
Action: re-check each flagged child's terminal outcome before any retry. Override
pass=false to pass=true only with the evidence chain (fires recorded by the
harness itself + duration >= deadline OR attempt>=2 with the full chain fired).
Record the cause as `deadline_kill`, not a quality failure.
Lesson: the watchdog's clock and the work's clock are different clocks; verdicts
come from the server's terminal outcome, not the harness's deadline.

## D.2 The inter-DC outage

Symptom: sync checks failing fleet-wide mid-run; every beat flat on one host.
Diagnosis: SYN arrives, SYN-ACK leaves the far end, never returns - inter-DC
return-path outage. Not a config defect; retries cannot fix routing.
Action: fail-open beats (log, exit 0), stand up a relay from a machine with a
working path, run the stalled step through the relay, log provenance. When the
direct path recovered, re-verified one relayed result directly.
Lesson: a watchdog that treats an outage as a child failure will kill healthy
children. Classify INFRA before KILL.

## D.3 The excerpt truncation false alarm

Symptom: scanner reported fields=0 for sessions the harvest measured at 14 fields.
Cause: the excerpt kept the first 4000 chars; the evidence sat in the last ~1500.
Fix: parse the full final text; when excerpting, anchor the tail at the evidence.
Watchdog rule: when two measurements disagree, the one that read the FULL artifact
wins; excerpt-based flags are UNVERIFIED until re-parsed.

# APPENDIX E - WATCHDOG SELF-HEALTH

The watchdog can be the broken thing. Every 10th beat, self-check:

1. beats.md grew this run (mtime/size changed).
2. The last beat's table has evidence in every child row (no all-UNVERIFIED table
   for 2 consecutive beats - that is the watchdog stalling, not the crew).
3. Corrections injected vs corrections verified are tracked; a watchdog with
   5 injections and 0 verified landings is injecting into the void - stop, read
   the parent's messages, and change the channel.
4. Context budget: beats.md holds the history; the session keeps only the last 3
   beats. A watchdog that fills its own context stops watching.

# APPENDIX F - GATE-LINE AND REPORTING CONTRACT

The watchdog does not emit its own gate line; it feeds the parent's:

```
gates -- ... | session-monitor: fired (beats N, corrections C, kills K)
```

"fired" = real beats with verified side-effects this turn. Reciting the beat
protocol from memory without arming the loop is "skipped: why", not fired - the
same honesty rule as every always-fire skill. The completion report (S7) and the
abort report (S8) are written to the runlog, not just chat, so the next session
resumes from files.

