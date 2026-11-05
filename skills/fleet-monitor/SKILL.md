---
name: fleet-monitor
description: Heartbeat and stall detection for long-running multi-seat fleet operations. Trigger on: any task running longer than 5 minutes, "monitor this run", "check if stuck", "is it still going", any multi-agent fan-out where seats run in parallel, any background job or overnight run. Includes: what constitutes a real heartbeat beat (verified side-effects), stall signal definitions, and recovery actions (scope absorption, re-spawn, escalation).
---

# Fleet Monitor

## Trigger

Arm this skill the moment any of these conditions are true:
- A task is estimated to take more than 5 minutes
- One or more subagents are running in background mode
- A shell command was launched without a timeout (background: true)
- A multi-seat fan-out has been dispatched (3+ parallel subagents)
- The user says "monitor this", "watch it", "let me know if stuck"

Never skip monitoring because you "think it will finish quickly." Terminals and background jobs freeze silently with no error output. The only way to know a job is alive is to verify a real side-effect.

## Instructions

### Phase 1 — Arm the Heartbeat

Set a mental timer: **check every ~3 minutes** (not 5, not 10 — 3 is the sweet spot between noise and late detection).

For each check, answer these three questions with evidence:

1. **Is the process alive?** (not "should be" — verify)
2. **Has new output been produced?** (file size grew, line count increased, new file appeared)
3. **Is the output sensible?** (no repeated error lines, no stuck loop)

### Phase 2 — Real Heartbeat Checks (use these, not assumptions)

**For shell commands / background jobs:**
```bash
# Check if process is still running (replace PID or name)
pgrep -f "my_command_name" && echo "ALIVE" || echo "DEAD"

# Check if output file is growing
wc -l /path/to/output.log

# Check last modification time of output
stat -c "%y %n" /path/to/output/* | tail -5

# Tail the last 10 lines of a log
tail -10 /path/to/output.log
```

**For subagents (background: true):**
- After dispatching, note the sessionID returned
- On each beat, check whether the task artifact (file, commit, log entry) has appeared
- A subagent that has not produced ANY file within 6 minutes of dispatch is a stall candidate

**For SSH remote jobs (VPS):**
```bash
ssh rabi@179.61.138.8 "pgrep -f 'opencode\|claude' | wc -l; ls -lt /srv/dr/projects/ | head -5"
```

**For file-producing tasks:**
```bash
# Count output files (should grow each beat)
ls /path/to/results/ | wc -l

# Confirm newest file is recent (within last 4 min)
find /path/to/results/ -newer /tmp/last_beat_marker -type f | wc -l
touch /tmp/last_beat_marker
```

### Phase 3 — Stall Signal Definitions

A run is STALLED when ANY of these are true:

| Signal | Definition |
|--------|-----------|
| FLAT_OUTPUT | Two consecutive beats show zero new files, zero log growth, zero new commits |
| PROCESS_DEAD | `pgrep` returns empty; SSH shows no matching process |
| ERROR_LOOP | Last 20 lines of log contain the same error pattern repeated 3+ times |
| TIMEOUT_EXCEEDED | Task has run >2x its estimated duration with <50% completion evidence |
| SILENT_HANG | Process is alive (pgrep shows it) but file output hasn't changed in 6+ min |
| SUBAGENT_TIMEOUT | Background subagent has produced no artifact after 8 minutes |

Two FLAT_OUTPUT beats in a row = confirmed stall. Act immediately.

### Phase 4 — Recovery Actions

**STALL → Recovery Ladder (try in order):**

1. **Diagnose first (30 seconds)**
   - Read the last 50 lines of the log
   - Check if the stall is at a specific step (which subagent? which file? which command?)
   - Is it waiting for input (stdin hang)? Is it a network timeout? Is it an OOM kill?

2. **Absorb scope** (if the stalled seat was doing something simple)
   - Cancel the stalled subagent
   - Complete its remaining work in the main session directly
   - Document what it had already produced

3. **Re-spawn with a sharper spec**
   - Kill the stalled process
   - Identify WHY it stalled (missing file? wrong path? rate limit?)
   - Fix the root cause in the spec
   - Re-launch with `background: true` and start monitoring immediately

4. **Escalate** (if re-spawn fails twice)
   - Write a STALL-REPORT.md: `symptom → root cause → what was completed → exact next step`
   - Notify the user with the report path
   - Do NOT keep re-spawning blindly — two stalls on the same task = a design problem, not a retry problem

### Phase 5 — Healthy Run Report Format

Every ~3-minute beat that shows a healthy run gets logged as one line:

```
[HH:MM] BEAT N — ALIVE | <N> files produced | last output: <filename> | ETA: <estimate>
```

At task completion:
```
[HH:MM] DONE — <N> total artifacts | runtime: <Xm Ys> | no stalls detected
```

If a stall was detected and recovered:
```
[HH:MM] RECOVERED — stall at beat 3 (FLAT_OUTPUT) | cause: <root cause> | recovery: <action> | resumed at beat 4
```

### Phase 6 — Multi-Seat Fan-Out Specific Rules

When 5+ subagents are running in parallel:

- Assign each seat a label (Seat 1..N) with its mandate and expected artifact
- Beat check = verify each seat's artifact exists OR the seat has returned
- A seat is stalled if it hasn't produced its artifact within 2x the median of other seats' completion times
- Never wait for a stalled seat to unblock the merge — absorb its scope or re-spawn it

## References

- Related skills: `tool-dispatcher` (right tool for each check), `result-classifier` (classify what comes back), `subagents` (fan-out rules), `godmode` (overall orchestration)
- VPS fleet: web-dev `179.61.138.8`, knowledge `179.61.251.98`
- Brain runbook: append stall root-causes to `cybersec/knowledge/` or `web-dev/knowledge/` as applicable

## Scripts

```bash
#!/bin/bash
# fleet_beat.sh — one-shot heartbeat check for a file-producing job
# Usage: bash fleet_beat.sh /path/to/output/dir "process_name"
OUTPUT_DIR="$1"
PROC_NAME="$2"

FILE_COUNT=$(ls "$OUTPUT_DIR" 2>/dev/null | wc -l)
ALIVE=$(pgrep -f "$PROC_NAME" > /dev/null 2>&1 && echo "YES" || echo "NO")
LAST=$(ls -t "$OUTPUT_DIR" 2>/dev/null | head -1)

echo "[$(date +%H:%M)] BEAT | alive=$ALIVE | files=$FILE_COUNT | last=$LAST"
```

```bash
#!/bin/bash
# vps_beat.sh — remote heartbeat check on the web-dev VPS
ssh rabi@179.61.138.8 "
  echo '=== PROCESSES ==='
  pgrep -a -f 'opencode\|claude\|node\|python' | head -10
  echo '=== RECENT FILES ==='
  find /srv/dr/projects/ -newer /tmp/last_vps_beat -type f 2>/dev/null | head -20
  touch /tmp/last_vps_beat
"
```