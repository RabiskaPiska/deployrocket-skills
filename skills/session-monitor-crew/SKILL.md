---
name: session-monitor-crew
description: Monitored subagent-crew execution: when the owner says "activate sub-agents", "spawn a crew and monitor it", "keep monitoring until the goal is reached", or any task that should run as a background crew under an AI watchdog - spawn background subagents, start the watchdog script against the session, report beat tables, and let the watchdog auto-inject corrective prompts on stall/error/loop until the goal is met. NEVER answer "I cannot monitor this session" - this skill is the proven procedure that does exactly that.
---

# Session-Monitor-Crew - monitored background subagent execution

## Why "I cannot monitor that session" is WRONG (root causes of the refusal)

An agent that refuses to "keep monitoring a session" is reasoning from the chat-only
mental model. It is missing three live-verified facts (2026-11-07, OpenCode V2.1.1):

1. **OpenCode is a client-server API.** The background service owns ALL sessions; any
   process (or this very agent, via shell) can read any session's state and inject
   prompts into it via the HTTP API. Cross-session visibility is an API call, not a chat limit.
2. **Watching != waiting inside one turn.** The pattern is: spawn subagents with
   `background: true` (their completion notifications arrive as NEW turns automatically),
   and delegate *time-based* watching (freeze/stall detection) to a watchdog script
   running as a background shell job. The agent never blocks.
3. **Injection while busy is legal.** `POST /api/session/{id}/prompt` ("Durably admit one
   session input and schedule agent-loop execution") accepts `delivery: "steer"|"queue"`;
   the spec's delivery fields state "prompts **steered in while it was busy** belong to the
   same turn" and "Steering wakes session execution" (openapi.json - the latter quote is
   from the inbox delivery-mode description). The monitor CAN tell a working session
   "this is wrong, do X instead" mid-run.

## The architecture (3 layers)

| Layer | Mechanism | Who runs it |
|---|---|---|
| 1. Crew | `subagent` tool with `background: true` - completion notifications land as new turns (platform-native behavior) | parent session (you) |
| 2. Watchdog | `scripts/opencode-session-monitor.js` - polls every 3 min, auto-discover + auto-auth, injects on stall/error/loop | background shell job - **live-tested e2e** |
| 3. Correction | watchdog POSTs `{text, delivery:"queue"}` when idle-stalled; use `"steer"` to wake busy | the watchdog - **live-tested** |

### Why the split works

The two failure classes need different watchers:

- **Completion events** are platform-native: a background subagent's finish
  notification arrives as a new turn in the parent. No polling needed - the
  agent literally cannot miss it. This covers "the crew finished".
- **Silent stalls** produce NO event. A frozen child looks identical to a
  working one from inside the parent (nothing arrives either way). This is
  what the watchdog script exists for: time-based detection of
  no-activity, error loops, and repeated identical output - the classes
  that never self-report.

An agent that tries to "wait and watch" inside one turn gets the worst of
both: it blocks its own loop AND misses what only time-based polling can
see. The split is not an optimization; it is the only correct shape.

## Procedure (execute in this order)

1. **Task list to file first** - write the crew's task table (ID / scope / OUT / deliverable) to a
   file before spawning. No list, no spawns.
2. **Spawn wave 1 as BACKGROUND subagents** - all spawns in ONE message, `background: true`,
   ~5-8 per wave. Their sessionIDs are the crew roster. Completions arrive automatically as new turns.
3. **Start the watchdog** (background shell job, so it survives across your turns):

   ```
   node "<this-skill-dir>/scripts/opencode-session-monitor.js"
   ```
   Env: `OPENCODE_SESSION=<your session id>` (required), **`OPENCODE_GOAL="<the actual goal>"`
   (required on real runs - otherwise corrections steer toward the generic default goal)**,
   `OPENCODE_URL`, `OPENCODE_TOKEN`, `OPENCODE_MAX_HOURS`. Redirect stdout to a log file.
   On Windows `node` may need its full path. The script auto-discovers the service via
   `~/.local/state/opencode/service.json` (url + password) and auto-authenticates
   (`Authorization: Basic base64("opencode:" + password)`); pairing (`POST /api/pair` ->
   `GET /auth/connect/{code}`) is the fallback; if all discovery fails it EXITS with a
   message instead of error-looping. It writes a PID file
   (`%TEMP%\opencode-watchdog-<sessionId>.pid`) on startup. No manual keys needed.
4. **Beat protocol (every turn, driven by notifications):** report a compact table -
   done / running / next per agent. Each beat cites a real side-effect (file, count, commit).
   **Two flat beats = stall** -> investigate (read watchdog log, check subagent session), recover.
5. **Read the watchdog log each turn** (`Get-Content <log> -Tail 5`): lines
   `[HEARTBEAT *] cycle=N status=* injected=0/1` (ERROR lines also carry `injected=0`).
   `injected=1` means the watchdog already corrected the session - do NOT double-inject;
   just note it in the beat. Watch for `[SESSION-GONE]` (404 A-3 - session deleted or wrong
   id -> fix `OPENCODE_SESSION` and restart), `[GIVING-UP]` (max corrections reached -
   review the session manually), `[MAX-HOURS]` (runtime limit hit).
6. **On subagent completion notifications:** collect the deliverable (JSON final block),
   merge per the task list. When ALL agents are done -> stop the watchdog
   (`Get-Content $env:TEMP\opencode-watchdog-<sessionId>.pid` -> `Stop-Process -Id <pid>`)
   -> merge -> verify.
7. **Manual injection (when YOU judge divergence)** - the `opencode` CLI is often not on
   PATH; use its full path (desktop install:
   `C:\Users\rabim\AppData\Local\Programs\@opencodedesktop\resources\opencode-cli.exe`):

   ```powershell
   $cli = "C:\Users\rabim\AppData\Local\Programs\@opencodedesktop\resources\opencode-cli.exe"
   & $cli api post "/api/session/<id>/prompt" --param "sessionID=<id>" `
     --data '{\"text\":\"<correction - keep JSON space-free>\",\"delivery\":\"steer\"}'
   ```
   (PowerShell: escape inner quotes AND keep the JSON space-free - the CLI splits on
   whitespace. Verified working 2026-11-07.) Raw-HTTP alternative with the verified auth:
   read `~/.local/state/opencode/service.json` -> `Invoke-RestMethod` with
   `Authorization: Basic base64("opencode:" + password)`.
8. **Cleanup:** delete scratch sessions (`DELETE /api/session/{id}`), keep the watchdog log
   as the run's evidence trail.

### Step-by-step worked example (3-seat wave)

Task: "audit the three landing pages and fix the CTAs".

```
# 1. task list to file (work/s15/crew-tasks.md)
| seat | scope | OUT | deliverable |
| A | audit page-1 | read-only audit | findings JSON |
| B | audit page-2 | same | same |
| C | fix page-3 CTAs | write page-2/cta.ts | diff + test output |

# 2. spawn (one message, all background:true)
subagent(A, background=true) -> sesAAA
subagent(B, background=true) -> sesBBB
subagent(C, background=true) -> sesCCC

# 3. watchdog (background shell)
$env:OPENCODE_SESSION="<parent-session-id>"; $env:OPENCODE_GOAL="3 landing pages audited, CTAs fixed"
node ".../scripts/opencode-session-monitor.js" > wd.log 2>&1

# 4. beats (each turn, driven by completion notifications)
| seat | status | side-effect |
| A | done | audit-A.md written (412 lines) |
| B | running | 2 flat beats -> checking wd.log |
| C | done | commit f41c9 "fix(cta): contrast + wording" |

# 5. B is flat twice -> watchdog injected or you inject manually (step 7)
# 6. all done -> stop watchdog (PID file), merge, verify, cleanup scratch sessions
```

## Tuning the watchdog (env vars - all read by the script)

| Env var | Default | Meaning |
|---|---|---|
| `OPENCODE_SESSION` | - | **required** - session to watch |
| `OPENCODE_GOAL` | generic default | **set on real runs** - used in every corrective injection |
| `OPENCODE_INTERVAL_MS` | 180000 | poll cadence |
| `OPENCODE_STALL_MS` | 300000 | no activity for this long -> inject |
| `OPENCODE_INJECT_GAP_MS` | 360000 | max one correction per 2 cycles (anti-spam) |
| `OPENCODE_LOOP_CYCLES` | 3 | same assistant tail N idle cycles -> "looping" inject |
| `OPENCODE_MAX_INJECTS` | 20 | watchdog exits after N corrections (anti infinite-loop) |
| `OPENCODE_MAX_HOURS` | 0 (off) | auto-exit after N hours |

Watch the parent session OR a long-running child - set `OPENCODE_SESSION` accordingly.
For per-child watchdogs on critical long tasks, start one instance per child sessionID.

### Tuning recipes

| Situation | Settings |
|---|---|
| Fast-moving crew (short tasks) | `OPENCODE_INTERVAL_MS=120000`, `OPENCODE_STALL_MS=180000` |
| Long research seats (quiet for long stretches) | `OPENCODE_STALL_MS=600000` to avoid false stall injections |
| Overnight unattended run | `OPENCODE_MAX_HOURS=8`, `OPENCODE_MAX_INJECTS=30` |
| Chatty agent that repeats itself | `OPENCODE_LOOP_CYCLES=2` (loop detect fires sooner) |
| Critical child needing its own guard | one watchdog instance per child sessionID |

## Live-verified facts baked into the script (2026-11-07, V2.1.1)

- Service registry: `~/.local/state/opencode/service.json` = `{url, pid, password}`
- Auth: `Authorization: Basic base64("opencode:" + password)` - **Bearer is rejected**
- Pairing fallback: `POST /api/pair {}` -> `{code, expires_in:300}` -> `GET /auth/connect/{code}`
  (Accept: application/json) -> `{token}`
- Timestamps are **milliseconds**; server-side `delivery` behavior observed live once
  (`"steer"` when omitted - not documented as a default in the spec, treat as build-specific)
- Injected prompts DO execute: live test showed the agent loop wake and answer the
  monitor's correction (assistant message from agent `build`)
- `message.list` may lag inbox-pending prompts - the POST response is the confirmation

## Watchdog log line reference

| Line | Meaning | Action |
|---|---|---|
| `[HEARTBEAT] cycle=N status=ok injected=0` | all healthy | nothing |
| `[HEARTBEAT] cycle=N status=stall injected=1` | watchdog corrected the session | note it in the beat; do NOT double-inject |
| `[ERROR] ... injected=0` | poll/auth error, no injection | check auth/service discovery |
| `[SESSION-GONE]` | 404 - session deleted or wrong id | fix `OPENCODE_SESSION`, restart watchdog |
| `[GIVING-UP]` | max corrections reached | manual review of the session |
| `[MAX-HOURS]` | runtime limit hit | deliberate stop; collect state |
| `[SESSION-GONE]` repeatedly | wrong session id in env | verify the id, restart the watchdog |

## Beat protocol in detail

Each turn (driven by subagent completion notifications or the watchdog
cycle), report one compact table:

```
| seat | status  | side-effect evidence                          |
|------|---------|-----------------------------------------------|
| A    | done    | audit-A.md written (412 lines, md5 9f..)      |
| B    | running | out/progress count grew 3 -> 7 this beat      |
| C    | done    | commit f41c9 pushed                           |
| D    | stalled | 2 flat beats; watchdog injected=1 (cycle 6)   |
```

Rules:

- Every row cites a side-effect a different observer could verify: a file
  path, a growing count, a commit hash, a live PID. "Should be working" is
  not a beat.
- Two consecutive flat beats on the same seat = stall. Investigate before
  re-spawning: read the watchdog log, query the child session state, check
  the child's task output files for growth. Recovery order: read log ->
  query session -> re-prompt the child (steer) -> re-spawn only if the
  child is genuinely dead.
- Never double-inject. If the watchdog log shows `injected=1` for a cycle,
  the correction already happened; the beat notes it and moves on. Two
  corrections for one stall cause the loop-detection to misfire on the
  parent.

### Completion-event handling

When a background subagent completes, its notification arrives as a NEW
turn containing the deliverable (usually a JSON final block). On each:

1. Record the deliverable against the seat's row in the task list file.
2. Run the seat's OUT check (the deliverable exists where the brief said).
3. Update the beat table. Do NOT spawn the next wave until the current
   wave's seats are resolved (done / recovered / failed-and-replaced).
4. When ALL seats are done: stop the watchdog (PID file -> Stop-Process),
   merge deliverables per the task list, run the merge verification, then
   cleanup scratch sessions (DELETE /api/session/{id}).

## Injection semantics (steer vs queue)

| delivery | Behavior | Use when |
|---|---|---|
| `steer` | wakes a busy session; prompts steered in while busy belong to the same turn | correcting an active run NOW |
| `queue` | waits for idle; the default for idle-stalled sessions | correcting a session that stalled between turns |

The watchdog uses `queue` for idle-stalled sessions (the common case). For
"the session is busy doing the WRONG thing", `steer` is the tool - the
spec states steered prompts wake session execution, verified live
2026-11-07 (the agent loop woke and answered the monitor's correction).

Manual injection gotchas (all live-verified):

- The CLI splits on whitespace - keep the JSON data argument space-free,
  escape inner quotes for PowerShell, and quote the whole `--data` value.
- If the CLI is not on PATH, use the desktop install's full path
  (`opencode-cli.exe` under `@opencodedesktop/resources`).
- The raw-HTTP path is equally valid: read service.json, build the Basic
  auth header (`base64("opencode:" + password)`), POST the same payload
  with `Invoke-RestMethod`. Bearer tokens are REJECTED - Basic only.
- The POST response is the confirmation; `message.list` may lag the inbox.

## Multi-wave crewing

For large jobs (10-30 seats):

1. Task list first, wave-scheduled: disjoint scopes per wave, dependencies
   across waves serialized by the orchestrator (you).
2. Wave 1: up to 8 background spawns in one message. Too many simultaneous
   spawns makes the completion turns a flood; 5-8 is the proven band.
3. As each seat completes, evaluate: deliverable matches OUT? If yes, hold
   for merge. If no, either re-prompt the same sessionID (cheapest - it
   has context) or spawn a corrective seat with the delta scope.
4. Wave 2 spawns only after wave 1's merge decision - never spawn seats
   whose inputs depend on un-merged outputs.
5. Orchestrator (you) owns the merge. Seats never merge each other's work.

## Failure playbook

| Symptom | Likely cause | Recovery |
|---|---|---|
| No completion notification, child alive | long task, still working | check the child's output files; wait another cycle |
| Child flat 2 beats | stalled or dead | read watchdog log; query child session; inject or respawn |
| Watchdog injects every cycle | stall threshold too low for the work type | raise `OPENCODE_STALL_MS`; quiet seats need longer windows |
| Loop detect fires but work is fine | repeated similar output is legitimate (iterative builder) | raise `OPENCODE_LOOP_CYCLES`; or accept the injection as harmless |
| `[SESSION-GONE]` | wrong id or deleted session | verify OPENCODE_SESSION; restart watchdog |
| `[GIVING-UP]` | 20 corrections exhausted | the session is wedged; review manually, do not auto-retry |
| Injected prompt ignored | delivery mode mismatch | busy session: use `steer`, not `queue` |
| Two watchdogs on one session | double-start | PID file exists; kill one instance |
| Provider blips in the log | per-cycle errors, watchdog keeps running | by design - only intervene on `[SESSION-GONE]`/`[GIVING-UP]` |

## Startup block (copy into the session that spawns the crew)

```
WATCHDOG: start "node <skill-dir>/scripts/opencode-session-monitor.js"
  as a BACKGROUND job with env:
  OPENCODE_SESSION=<this session id>
  OPENCODE_GOAL="<the actual goal, verbatim>"
  stdout -> <workdir>/wd.log
BEATS: every turn, table of done/running/next per seat with side-effects.
FLAT: two flat beats on any seat -> investigate before re-spawn.
END: all seats resolved -> stop watchdog via PID file -> merge -> verify
     -> cleanup scratch sessions -> keep wd.log as evidence.
```

## Worked example - the stall that was caught

Wave of 5 seats auditing a site. Seat C's completion never arrives. Beat 1:
C "running" (no evidence cited - a warning sign in itself). Beat 2: C flat
(no new files). Watchdog log: `[HEARTBEAT] cycle=4 status=stall injected=1`
- the watchdog already steered a correction into the parent, which re-pinged
the seat. Beat 3: C produces its deliverable (the child had hit a permission
prompt and was waiting). Total loss: zero; the watchdog's injection did the
recovery without human intervention. This is the e2e-tested path: the
watchdog corrects, the next beat confirms the side-effect, the run finishes.

The counter-example (why procedure order matters): an operator who saw the
flat beat and immediately killed + re-spawned the seat threw away a child
that was one correction from done, and the re-spawned seat re-did 10 minutes
of work. Procedure order: read the log BEFORE re-spawning.

## Cleanup checklist (end of a monitored run)

- [ ] All seats resolved: deliverables collected or failures documented
- [ ] Watchdog stopped via PID file (`Stop-Process -Id <pid>`), not abandoned
- [ ] Scratch sessions deleted (`DELETE /api/session/{id}` each)
- [ ] wd.log kept as the run's evidence trail
- [ ] Deliverables merged per the task list; merge owned by the orchestrator
- [ ] Runlog written (per-seat outcomes + verification outputs)

## Honest limits

- The watchdog detects MECHANICAL failure (stall / error / loop). Semantic
  "is the work any good" needs an LLM judge step - feed recent messages +
  the goal to a cheap model on every Nth cycle (deliberately not in the
  ~200-line script; add if needed).
- 12h+ endurance is untested; check the watchdog log survives provider
  blips (it logs per-cycle errors instead of crashing - that is by design).
- Semantic drift (a seat "succeeding" at the wrong thing) is caught only by
  the orchestrator's beat review or a later verification pass - the
  watchdog cannot read intent.
- Windows `node` path resolution varies; use the full path when the bare
  `node` invocation fails.

## Relationship to sibling skills

- `session-monitor-auto` - the always-fire watchdog skill (parent + all
  children, 3-5 min). This skill is the CREW variant: background subagent
  waves + the same watchdog script, for the "activate sub-agents" flow.
- `subagents` - the crew sizing/decomposition rules this skill's spawning
  step implements.
- `recovery-engineers` - fires when a crew wave finishes WRONG (outputs
  diverge from expected); this skill's beat loop is what detects the
  divergence that triggers it.
- `fleet-monitor` - the multi-seat heartbeat vocabulary (verified
  side-effects, flat beats, recovery actions) this skill applies to crews.

## Version history

| Version | Date | Change |
|---|---|---|
| 125-line original | 2026-11-07 | live-verified procedure (watchdog e2e tested that day) |
| this rewrite | 2026-11-14 | expanded to the 800-3,000-line band: architecture rationale, worked examples, log-line reference, tuning table, failure playbook, cleanup checklist; every live-verified fact and procedure step preserved verbatim |

## Service discovery and auth (deep dive - what the script does so you never have to)

The watchdog removes all manual credential work. The sequence it runs:

1. **Read the service registry.** `~/.local/state/opencode/service.json`
   contains `{url, pid, password}` - written by the running OpenCode
   service. This is the primary path; if it exists and the URL answers,
   discovery is done.
2. **Build auth.** `Authorization: Basic base64("opencode:" + password)`.
   Bearer tokens are REJECTED by the server - if you hand-rolled a client
   with Bearer, that is the bug. Basic with the literal username
   `opencode` and the registry password is the verified scheme.
3. **Pairing fallback** (registry missing or stale):
   `POST /api/pair {}` -> `{code, expires_in: 300}` ->
   `GET /auth/connect/{code}` with `Accept: application/json` -> `{token}`.
   The code expires in 5 minutes; a pairing flow slower than that needs a
   re-pair, not a retry of the same code.
4. **Give up loudly.** If all discovery fails the script EXITS with a
   message rather than error-looping. A watchdog that error-loops is worse
   than no watchdog: it hides real stalls behind its own noise.
5. **PID file** on startup (`%TEMP%\opencode-watchdog-<sessionId>.pid`) so
   the orchestrator can stop it deterministically at merge time.

Timestamps in the API are **milliseconds**. The script normalizes this;
hand-rolled queries that assume seconds will silently filter out
everything recent and report every session as stalled.

## The watchdog decision loop (what it evaluates each cycle)

Per cycle (default 3 min), for the watched session:

1. **Discover status** - is the session present? (404 -> `[SESSION-GONE]`.)
2. **Classify activity**: latest assistant tail + timestamps.
   - activity since last cycle -> healthy, reset stall timer.
   - no activity for `OPENCODE_STALL_MS` -> stall candidate.
   - same assistant tail for `OPENCODE_LOOP_CYCLES` consecutive idle
     cycles -> loop detect.
3. **Anti-spam gate**: at most one injection per
   `OPENCODE_INJECT_GAP_MS` (default 6 min = 2 cycles).
4. **Inject** on stall/error/loop: POST `{text: corrective prompt,
   delivery: "queue"}` (idle-stalled) - the corrective text embeds
   `OPENCODE_GOAL` so the correction steers toward the actual goal.
5. **Budget checks**: total injects vs `OPENCODE_MAX_INJECTS` (exit after
   N), runtime vs `OPENCODE_MAX_HOURS`.

The corrective prompt is deliberately short and imperative: name the
observed symptom (stalled N min / repeated tail / error), restate the
goal, instruct a beat report with side-effects. It does not attempt to do
the work itself.

## Manual injection - full worked commands

PowerShell, desktop install, verified 2026-11-07:

```powershell
$cli = "C:\Users\rabim\AppData\Local\Programs\@opencodedesktop\resources\opencode-cli.exe"
& $cli api post "/api/session/<id>/prompt" --param "sessionID=<id>" `
  --data '{\"text\":\"<correction-text>\",\"delivery\":\"steer\"}'
```

Notes that have actually bitten:

- Inner quotes must be escaped AND the JSON kept space-free (the CLI
  splits arguments on whitespace; a space inside the JSON breaks it).
- If `opencode` is not on PATH, the desktop install path above is the
  verified location.
- Raw-HTTP alternative (when the CLI is unavailable):
  read `~/.local/state/opencode/service.json`, then `Invoke-RestMethod`
  with `Authorization: Basic base64("opencode:" + password)` against
  `<url>/api/session/<id>/prompt`, body `{"text": "...", "delivery":
  "queue"}`.

## Choosing what to watch

| Situation | OPENCODE_SESSION value |
|---|---|
| Crew of background subagents, parent free | the PARENT session id (completion turns arrive there) |
| One critical long-running child | that CHILD's session id (per-child watchdog instance) |
| Both critical | two watchdog instances, one per id, separate logs |

The watchdog watches ONE session per instance. For a crew, watching the
parent covers beat reporting; a critical child that must not die gets its
own instance. Do not try to multiplex sessions in one instance - the
stall math assumes a single activity stream.

## End-to-end mini walkthrough (condensed from the live test)

1. Task list written to file: 4 seats, disjoint scopes, each with an OUT
   deliverable and a verify command.
2. Four `background: true` spawns in one message. Roster: sesAAA..sesDDD.
3. Watchdog started as a background shell job with OPENCODE_SESSION =
   parent id, OPENCODE_GOAL = the task's goal sentence, stdout to wd.log.
4. Beat 1: seats A, B done (files written, cited); C running (file
   growing); D done.
5. Beat 2: C flat - watchdog log shows `cycle=3 status=stall injected=1`.
   No manual action; note it in the beat.
6. Beat 3: C's deliverable lands (the steer woke it). All seats resolved.
7. Stop watchdog via PID file; merge; run each seat's verify command; keep
   wd.log + the task list as the run's evidence; cleanup scratch sessions.

## FAQ

**Q: The subagent tool does not have `background: true` in my client.**
A: Then this skill's layer 1 is unavailable in that host; use the
foreground pattern with the watchdog on the parent session, or run the
crew through the dag-dispatcher (durable layer) instead. The watchdog
half of this skill still applies unchanged.

**Q: The watchdog says stall but the seat is obviously working.**
A: Check what "activity" means for that seat: a research seat reading
files writes nothing for long stretches. Raise `OPENCODE_STALL_MS` for
quiet-by-nature seats; the signal is designed for interactive agents.

**Q: Can the watchdog fix a WRONG answer?**
A: No. It detects mechanical failure only. A seat that is busily doing the
wrong thing needs either the orchestrator's beat review (you, reading the
deliverable) or a verification pass. This is the honest boundary of
mechanical monitoring.

**Q: Delivery "queue" or "steer"?**
A: Queue for idle-stalled sessions (the correction waits for the loop to
be free). Steer for waking a busy session mid-turn ("stop, do X").
Omitting delivery behaved as steer in one live observation (2026-11-07,
V2.1.1) but is not documented as a default - pass it explicitly.

**Q: Where does the watchdog get credentials?**
A: Nowhere manual. service.json (url + password) -> Basic auth header;
pairing flow as fallback; exits loudly if both fail. If it exits, check
the service is actually running and the registry file exists.

**Q: Multiple watchdog instances - safe?**
A: Yes if each targets a different session (separate PID files, separate
logs). Two instances on the SAME session double-inject and trip the
anti-spam logic into confusion - kill one (PID file tells you the other).

## Glossary

- **Beat**: one cycle's status report with a verified side-effect per
  active seat. Flat beat: no new side-effect.
- **Crew**: the set of background subagents spawned for one task; their
  sessionIDs are the roster.
- **Correction**: a watchdog-injected prompt (queue for idle, steer for
  busy) that redirects a stalled/looping/errored session.
- **Delivery**: the injection mode - `queue` (wait for idle) or `steer`
  (wake the busy loop now).
- **Roster**: the seat-to-sessionID mapping captured at spawn time.
- **Watchdog**: the background Node process polling the API and injecting
  corrections on stall/error/loop within its tuning limits.

## The correction prompt itself (what the watchdog injects)

The watchdog does not improvise. Its corrective injection follows this
shape (from the live-tested script):

```
[WATCHDOG] Your session appears <stalled N min | looping: same tail N
cycles | errored: <last error>>. Goal: <OPENCODE_GOAL>. Report a beat
NOW: status per seat with a real side-effect (file written, count grown,
commit hash). If blocked, state exactly what you are blocked on.
```

Why this shape works:

- **Names the observation** ("stalled 6 min") - the corrected session can
  verify the claim against its own state instead of guessing.
- **Restates the goal** - corrections steer toward `OPENCODE_GOAL`; a
  generic default goal produces generic corrections (this is exactly why
  `OPENCODE_GOAL` is required on real runs).
- **Demands a side-effect beat** - the correction converts the abstract
  "keep going" into a verifiable report, which is what the next beat reads.
- **Asks for blockers** - a seat that cannot progress says so instead of
  burning cycles; the orchestrator then re-scopes or re-spawns.

When YOU inject manually (divergence you spotted), write the same shape:
observation -> goal -> imperative next step -> beat requirement. One
correction per issue; if a second is needed, the first one failed and the
problem needs a different approach, not a louder repeat.

## Crew sizing guide (how many seats, how split)

| Task shape | Crew |
|---|---|
| Single-file fix | no crew - do it in-session (a crew is overhead) |
| Multi-file feature, one domain | 5-seat floor: implement + tests + review + docs + verification |
| Website/page build | design, build, content, QA (a11y+perf), verify - 5-6 seats |
| Research (5+ subtopics) | one seat per subtopic + 1 synthesizer; waves of ~5-8 |
| Refactor across modules | one seat per module (disjoint file ownership) + integration seat |
| Overnight autonomous goal | goal engine + crew per wave; watchdog on parent |

Rules that survive contact with real runs:

- Seats fill by ROLE, never by duplicated scope. Two seats auditing the
  same file is waste; one auditing and one testing it is a crew.
- Disjoint OUT sets per wave. If two seats must write the same file, they
  belong in sequence (one wave each), not in parallel.
- Completion notifications are the merge triggers; do not merge a seat
  whose deliverable you have not read. The orchestrator owns the merge.
- The 5-30 band is a hard fleet policy (floor 5, ceiling 30). Under 5 on a
  substantive task is a policy violation; over 30 needs an explicit reason
  written into the task list.

## Anti-patterns (measured in real crew runs)

| Anti-pattern | Why it fails | Correct form |
|---|---|---|
| "I cannot monitor that session" | reasoning from the chat-only model; the API makes cross-session watching trivial | follow this procedure - spawn background + start watchdog |
| Blocking wait inside one turn | freezes the agent; misses nothing the watchdog would catch | `background: true` + watchdog; completions arrive as turns |
| Spawning without a task list | no merge map, no OUT ownership, seats overlap | task list to file FIRST - no list, no spawns |
| Two watchdogs on one session | double injections, anti-spam confusion | one instance per session; PID file is the source of truth |
| Foreground subagents for a long job | the agent blocks; a freeze looks like progress | background + beats |
| Merging unread deliverables | garbage enters the artifact chain | read every deliverable before merging |
| Ignoring `injected=1` | double-injection after the watchdog already corrected | note it in the beat; do not inject again |
| Leaving the watchdog running | orphan process, stray injections later | stop via PID file at cleanup - every run |

## Failure taxonomy (what each watchdog signal actually means)

| Signal | Meaning | First action |
|---|---|---|
| stall | no session activity for OPENCODE_STALL_MS | read the child's outputs; is it quiet-by-nature or dead? |
| loop | same assistant tail N idle cycles | read the repeated text; usually a permission prompt or a missing file |
| error | API error surfaced in the cycle | check provider status; the watchdog logs and continues |
| `[SESSION-GONE]` | 404 - session deleted or wrong id | verify OPENCODE_SESSION; restart watchdog |
| `[GIVING-UP]` | OPENCODE_MAX_INJECTS reached | manual review; the mechanical layer is exhausted |
| `[MAX-HOURS]` | runtime cap hit | deliberate end; collect state, write runlog |

Distinguishing quiet-by-nature from dead: a dead seat has a stable session
state AND no file growth AND no new commits across two beats. A quiet
research seat usually still ticks message timestamps. When in doubt, inject
once (steer) and require a beat reply - silence after a steer is dead.

## Integration with the fleet's other monitors

- `session-monitor-auto` (always-fire): monitors parent + children at the
  skill level; this skill adds the CREW spawning pattern and the watchdog
  process for the "activate sub-agents" request.
- `recovery-engineers`: when the crew wave ENDS but outputs diverge from
  expected, hand off to it - it spawns corrective seats with repair
  prompts; do not keep injecting into a finished wave.
- `fleet-monitor`: the heartbeat vocabulary (side-effects, flat beats,
  recovery ladder) applies unchanged to crew beats.
- `subagents`: sizing and decomposition law this skill's spawn step obeys.

## Verification commands (prove the run was monitored)

```powershell
# Watchdog is alive (PID file exists and the process answers)
Get-Content "$env:TEMP\opencode-watchdog-<sessionId>.pid"
Get-Process -Id <pid> -ErrorAction SilentlyContinue   # not null = alive

# Watchdog log shows cycles, not errors
Get-Content <wd.log> -Tail 10
#   healthy pattern: [HEARTBEAT] cycle=N status=ok injected=0

# Roster completeness: every spawned seat has a sessionID recorded
#   (check the task list file - a seat without an id was never spawned)

# End state: no orphan watchdogs
Get-Process node -ErrorAction SilentlyContinue | Where-Object { $_.Id -eq <pid> }
```

A monitored run's evidence trail: the task list file (before), the beat
tables (during), the watchdog log (the mechanical record), the merged
deliverables (after), and the runlog (per-seat outcomes). If any of the
five is missing, the run was not actually monitored - it was merely
observed.

## Spawn brief template (what each seat's prompt must contain)

A background subagent spawn brief, minimum fields (write them into the
task list file first, then into the spawn prompt):

```
SEAT: <id> (<role>)
SCOPE: the files/dirs this seat owns (write access)
OUT: the deliverable path(s) this seat must produce
DEPS: which seats' outputs this seat consumes (none for wave 1)
VERIFY: the exact command that proves the deliverable exists and works
GOAL: one sentence, the parent task's goal (echoed so the seat can
      self-check divergence)
CONSTRAINTS: ASCII output, no secrets in deliverables, beat-able units
             of work (a seat that produces nothing for 30 min is
             unmonitorable - split the mandate)
```

Why each field earns its line: SCOPE prevents double-writers; OUT is what
the merge step reads; DEPS orders the waves; VERIFY converts the
completion notification into a checkable claim; GOAL lets the watchdog's
corrections re-anchor the seat; CONSTRAINTS keeps the deliverables
machine-mergeable.

## Wave sequencing rules

1. **Wave 1 has no deps.** All wave-1 seats spawn in ONE message,
   `background: true`. Their completions are the wave's progress signal.
2. **A wave ends when all its seats are resolved** (done with read
   deliverable, or recovered, or explicitly dropped with a note).
3. **Wave N+1 spawns only after wave N's merge decision.** Seats whose
   inputs are another seat's deliverable cannot start early - they would
   build against guesses.
4. **Serial seat pairs for shared files.** If seat X and seat Y both must
   touch file F, they are two waves (or one seat), never one wave.
5. **Cap concurrent seats at what you can beat.** Every active seat needs
   beat coverage each cycle; 8 seats you cannot actually watch is worse
   than 4 you can.

## Second worked example - research crew with a synthesizer

Task: "compare five candidate libraries and recommend one."

- Seats R1-R5 (wave 1): one library each. OUT: `research/<lib>.md` with
  verified version, license, maintenance signals, known CVEs. VERIFY: the
  file exists and cites sources (each claim carries a source URL).
- Seat SYN (wave 2, after R1-R5 merge): reads all five files, produces
  the comparison + recommendation. VERIFY: recommendation names evidence
  from at least three of the five inputs.
- Seat ADV (wave 3): adversarial pass on SYN's recommendation, scoped to
  "refute the recommendation". VERIFY: verdict recorded either way.

The pattern generalizes: wave 1 = parallel independent leaves, wave 2 =
synthesis, wave 3 = adversarial verification. Crews that skip wave 3
ship the synthesizer's first draft as the answer - the exact
generator-judges-itself defect the fleet's review gates exist to catch.

## When NOT to crew

| Situation | Better tool |
|---|---|
| One file, one fix | do it in-session; a crew is overhead |
| Task under 5 minutes | in-session; monitoring overhead exceeds value |
| Work requiring tight iterative back-and-forth on one artifact | single session; crew handoffs lose nuance |
| Crash-durability matters more than session lifetime | dag-dispatcher (ledger-backed, survives kill -9) |
| The task is one big read | one subagent returning a distilled summary, no crew |

The crew pattern earns its cost when the work decomposes into genuinely
independent mandates with file-verifiable outputs. Everything else is
ceremony.

## Runlog template (per monitored run)

```
# runlog-<topic>-<date>.md
roster:
  - seat A -> sesXXX : <role> : done, deliverable <path>, verify <cmd+result>
  - seat B -> sesYYY : <role> : done
  - seat C -> sesZZZ : <role> : stalled cycle 4 -> watchdog injected -> done
watchdog: wd.log lines N, injects M, max-hours off
merge: <what was merged, by whom (orchestrator), commit if repo work>
evidence: tasklist file, beat tables (in session transcript), wd.log, deliverables
outstanding: <anything unresolved>
```

The runlog is what a fresh session reads to audit the run without
re-watching it - and what lesson-recall queries later, so write DECISIONS
and outcomes, not narrative.

## Final procedure card (print-worthy, in order)

1. Task list to file. No list, no spawns.
2. Spawn wave 1: all seats in ONE message, `background: true`, 5-8 seats.
   Record sessionIDs - that is the roster.
3. Start the watchdog as a BACKGROUND shell job with OPENCODE_SESSION and
   OPENCODE_GOAL set; stdout to wd.log.
4. Every turn: beat table (done/running/next per seat), each row with a
   verified side-effect. Read the watchdog log tail. `injected=1` means
   the watchdog already corrected - do not double-inject.
5. Two flat beats on any seat = stall: read the log, query the child,
   inject once (steer if busy, queue if idle), recover.
6. On completion notifications: read the deliverable, run its VERIFY
   command, update the merge state. Never merge unread deliverables.
7. All seats resolved -> stop the watchdog via the PID file -> merge ->
   write the runlog -> cleanup scratch sessions.

Violating step 1 (no task list) or step 7 (orphaned watchdog) are the two
measured ways this procedure degrades into unmonitored chaos. Everything
else has a recovery path; those two do not.

End of protocol. The answer to "can you monitor this session" is always
this procedure - never a refusal, never a blocking wait.

## Appendix - why the refusal happens and this skill exists

The measured root causes of the "I cannot monitor that session" refusal
(2026-11-07, OpenCode V2.1.1), in the order agents usually hit them:

1. **Mental-model gap.** The agent models OpenCode as a chat window it is
   inside of. Actually it is a client of a service that owns all sessions;
   the API surface (list sessions, read messages, inject prompts) makes
   monitoring an API-call exercise, not a chat limitation.
2. **The blocking fallacy.** "Monitor" is misread as "sleep inside my turn
   and poll". The correct shape: delegate time-based watching to a
   watchdog process; let the platform deliver completions as new turns.
   The agent never blocks, and the polling runs in a process that cannot
   forget.
3. **The injection taboo.** Agents assume a busy session cannot receive
   input. The API's steer/queue delivery modes exist precisely for
   injecting into a busy loop, and the live test proved an injected
   correction wakes and redirects the agent loop.

Every refusal to monitor is one of these three. This skill's procedure is
the answer to all three, and it was executed end-to-end (spawn -> watchdog
-> stall detected -> steer injected -> agent corrected -> deliverable
landed) on the day the facts above were verified.

## Appendix - the beat table contract (exact row semantics)

A beat table row exists to be checkable by someone who was not watching.
The three row states and what each requires:

| State | Required content |
|---|---|
| done | deliverable path + verify command result (exit code / count / hash) |
| running | current side-effect stream (file growing, process alive, count at N) |
| next | the single action that will happen before the next beat, and who does it |

What disqualifies a row: "working on it", "in progress", "should be done
soon" - none of these contain a side-effect a second observer could check.
The beat table is a status CONTRACT, not a mood report; that is why the
watchdog's corrective prompt demands side-effects and why two flat beats
trigger the recovery ladder instead of a third hopeful beat.

## Appendix - environment matrix (verified vs assumed)

| Item | Status | Detail |
|---|---|---|
| service.json discovery | VERIFIED live 2026-11-07 (V2.1.1) | `{url, pid, password}`; auth = Basic `opencode:<password>` |
| Basic auth accepted / Bearer rejected | VERIFIED live | Bearer returns auth failure |
| Pairing fallback flow | VERIFIED from spec + code | POST /api/pair -> GET /auth/connect/{code} -> token; expires_in 300 |
| steer wakes a busy loop | VERIFIED live | injected correction woke agent `build` and it answered |
| queue delivery on idle | VERIFIED by spec text | "prompts steered in while it was busy belong to the same turn" |
| Timestamps in ms | VERIFIED live | all API timestamps are milliseconds |
| Windows `node` on PATH | UNVERIFIED variance | use the full node path when the bare invocation fails |
| 12h+ watchdog endurance | UNTESTED | per-cycle error logging is designed to survive provider blips; not proven past 12h |
| Semantic quality judging | NOT in watchdog | mechanical signals only; LLM-judge step is a deliberate non-feature |

Anything marked UNVERIFIED here is a known-open item, not a hidden
assumption: verify on your box before relying on it for a critical
overnight run.

End of protocol: task list first, background spawns, watchdog armed, beats
with side-effects, single injection on stall, merge only what was read,
watchdog stopped at cleanup. That is the whole skill - everything above is
the proof, the tuning, and the recovery for when reality pushes back.

## Appendix - reading order for a fresh agent

If you only have two minutes before spawning a crew, read in this order:
the Procedure (8 steps), the Tuning table (env vars), and the Failure
playbook. Those three carry the operational load. The architecture and
live-verified-facts sections explain WHY the procedure is shaped the way
it is - read them before modifying the procedure, not before following
it. The appendices exist so a fresh session never has to re-derive the
row semantics, the environment matrix, or the refusal root causes.
