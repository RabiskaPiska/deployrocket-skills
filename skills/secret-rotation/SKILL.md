---
name: secret-rotation
description: Rotate API keys and credentials across ALL fleet config roots (including dual-root boxes) with zero-downtime handoff - new key deployed beside the old, grace window, consumers verified with the new key (ack), old key revoked only after every consumer acknowledged, append-only audit log. Fires on credential exposure (any leak signal), new-provider onboarding, and the 90-day cadence. Closes the ai-jury plaintext-key class (live keys hardcoded in a publicly fetchable SKILL.md) and the "rotation on human memory" anti-pattern.
alwaysFire: false
---

# secret-rotation -- Fleet API Key Rotation with Zero-Downtime Handoff

## 0. PURPOSE

Three measured facts motivate this skill:

1. The ai-jury audit finding (skill-audit-batch1): **live API keys hardcoded in
   plaintext** inside a publicly fetchable SKILL.md (`EL_KEY=xpl_42e8...`,
   `AR_KEY=ar_al3a...`) -- secrets in a file the catalog serves to anyone. The class
   is not hypothetical; it shipped.
2. GitGuardian (via the 2026-08-05 Augment Code pipeline guide, verified in
   runlog-RGAP2): "agents consume credentials constantly and leak them at elevated
   rates"; rotation "has to run on a schedule or a lease, never on human memory."
3. Campaign reality: passwords and keys rotated repeatedly mid-run (serve password
   rotated twice in one run; provider swaps x3+), and every rotation without a
   handoff protocol broke downstream consumers until they re-read the new secret.

This skill is the procedure for rotating ANY fleet credential with ZERO downtime and
ZERO silent breakage: deploy-new -> grace -> ack -> revoke-old -> audit.

## 1. WHEN TO FIRE

Fire on ANY of:

- **Exposure**: a secret appeared in a repo, transcript, skill file, log, screenshot,
  or was echoed in a chat (the ai-jury class). Fire IMMEDIATELY -- rotation-first,
  cleanup-second, always in that order.
- **New provider onboarding**: first credential for a new provider lands in the
  fleet (it must enter through this skill's inventory + storage rules, not through
  a loose paste into one config file).
- **90-day cadence**: scheduled rotation. The cadence lives in a cron/systemd timer
  or the goal scheduler -- NEVER in human memory (GitGuardian's rule, adopted
  verbatim).
- **Personnel/credential-scope change**: a seat loses access, a provider key is
  shared to a new box, a contractor/subagent mandate ends.
- **Suspected compromise** even without proof: an unexplained 401 from somewhere
  you did not touch, a canary that starts failing with auth errors, an unknown IP
  in a provider dashboard.

### Fire-model justification

Situational (not always-fire): rotation is an event-driven procedure, not a
per-turn gate -- firing it every turn would be pure token waste (RGAP2's own
verdict for credential rotation: scheduled job + hooks for scanning, not an
always-fire body). BUT the *triggers* above are hard triggers: exposure fires it
the same turn the exposure is discovered, no scheduling, no confirmation round.

## 2. INVENTORY -- MAP EVERY KEY LOCATION FIRST (read-only, always first)

You cannot rotate what you have not enumerated. The fleet's known key locations:

| Location class | Examples (this fleet) | Rotation touchpoint |
|---|---|---|
| Config roots, primary | `~/.config/opencode/config.jsonc` per box | provider blocks, `{file:}` key refs |
| Config roots, dotroot | the second root on dual-root boxes | same fields; the drift trap |
| Standalone key files | `/home/rabi/.keys/*.key` (0600) | the file CONTENT is the rotation target |
| Skill/agent files | SKILL.md frontmatter, env exports in examples | the ai-jury class: inline `KEY=...` lines |
| Serve/runtime logs | `/tmp/oc-serve*.log` (rotated passwords!) | consumers re-read from the NEWEST log |
| Provider dashboards | key lists, PATs per provider | create/revoke happen here; dashboards are the only source of truth for "revoked" |
| Session/plugin state | per-seat provider pins, cached tokens in sessions | pins die with the old key; revalidate |
| CI/env files | `.env`, MCP configs, `claude mcp` per config root | per-root, never per-machine (the mcp-install-is-per-config-root lesson) |

Inventory commands (paste output into the rotation runlog):

```bash
# All key-ish references in config roots (both roots, every box):
grep -rn -iE "api_key|apikey|authorization|token|secret" \
  ~/.config/opencode/ <dotroot>/ 2>/dev/null | grep -v ".pre-swap"

# Inline-literal scan (the class this skill closes): any long literal in configs/skills:
grep -rnE "[A-Za-z0-9_]{8,}=(sk|xpl|ar_|sk-ant|hf_)[A-Za-z0-9_-]{8,}" \
  ~/.config/opencode /srv/skills 2>/dev/null

# Key-file perms:
find /home/rabi/.keys -type f -exec stat -c "%a %n" {} \;   # expect 600 each
```

Any literal found by the inline scan is BOTH a rotation trigger AND a leak to scrub
(rotation first: the literal is burned the moment it was written anywhere).

## 3. THE ROTATION PROCEDURE (zero-downtime, five phases)

### Phase A -- PREPARE (no traffic touched)

```
A1 Inventory (section 2) -> the KEY MAP: every location of the old secret,
   every consumer (which box/root/process/seat reads it), every provider binding.
A2 Create the NEW key in the provider dashboard. Name it with a date:
   <purpose>-2026-11-12-r2 (names make the audit log self-describing).
A3 Write the new key to a NEW file: /home/rabi/.keys/<provider>-2026-11-12.key
   mode 0600 (Windows: ACL to the service user only). NEVER echo it to the
   terminal, transcript, or chat -- handle by path.
A4 Prove the NEW key works BEFORE touching any config: models 200 + one real
   completion (same probes as provider-failover section 6). A new key that cannot
   complete must not proceed to deploy.
```

### Phase B -- DEPLOY BESIDE (the zero-downtime core)

```
B1 For every location in the key map, make the config accept BOTH keys for the
   transition. Patterns, best first:
   - Dual-key window: if the provider config format allows a key LIST, put
     [new, old] -- providers that accept multi-key lists give true zero-downtime.
   - Grace by ordering: update consumers in a staged order so no consumer ever
     references only the dead key (see C ordering below).
   - If the provider allows only one active key: prepare the new key file but DO
     NOT cut over until every consumer is ready to re-read within seconds
     (single-box case), or schedule the cutover in a maintenance beat (multi-box).
B2 Apply to ALL roots on ALL boxes (dual-root law). One root updated = the secret
   is now inconsistent, which is worse than unrotated (two different live keys,
   one of them unknown to the audit).
```

### Phase C -- CUTOVER (the only moment downtime is possible; make it one beat)

```
C1 Order consumers by blast radius: canary consumer first, then the long-running
   seats, then batch/cron consumers.
C2 Flip each consumer to the new key: update {file:} references (never inline the
   value), restart/hot-reload the consumer, verify with the completion probe.
C3 ACK each consumer: it has made ONE successful authenticated call with the NEW
   key. An ack is a real side-effect (HTTP 200 + plausible body), never "config
   looks right".
```

### Phase D -- GRACE WINDOW (overlap, on purpose)

```
D1 Keep the OLD key VALID during the grace window (provider-permitting):
   default grace = 15 min (config-plane swaps) up to 24h (dashboards you cannot
   cut over atomically; say the actual number in the runlog).
D2 During grace: watch for 401/403 on the NEW key (deployment mistake) and for
   continued success on the OLD key (a consumer you missed).
D3 Any consumer still succeeding on the OLD key at the end of grace = missed
   inventory entry. Update the map (section 2), extend grace, re-run C.
```

### Phase E -- REVOKE AFTER ACK (the safety inversion)

```
E1 Revoke the OLD key ONLY after: every consumer in the key map has ACKed on the
   new key AND grace has elapsed.
E2 Revoke in the provider dashboard; then PROVE the revocation: the OLD key must
   now get 401. A rotation that never revokes is a key-duplication exercise.
E3 If a consumer breaks at revoke time despite ack: that consumer had a hidden
   copy of the old key -- the inventory (section 2) missed a location. Find it via
   the 401's source, add it to the map, rotate THAT consumer to new, keep grace
   open until it acks. (This is why ack-before-revoke is the invariant, not a
   suggestion.)
```

### Phase F -- AUDIT LOG (append-only, no secret values)

`<goal-dir>/secret-rotation-audit.log` (append-only JSONL):

```
{"ts":"2026-11-12T14:02:11Z","provider":"el","action":"created","key_name":"el-2026-11-12-r2","by":"session-abc","evidence":"probe 200 completion 200"}
{"action":"deployed","locations":["vps1:~/.config/opencode","vps1:dotroot","vps2:both-roots"],"probe":"200/ok"}
{"action":"acked","consumer":"vps1:opencode-serve","evidence":"completion 200 @09:03:22Z"}
{"action":"revoked_old","verified":"401 on old key @09:18Z"}
{"action":"scrubbed","target":"/srv/skills/skill-ai-jury/SKILL.md","evidence":"grep -c xpl_ == 0"}
```

Rules: timestamps in UTC, key NAMES not values, one line per state change, never
truncate the file. The audit log is the artifact the next rotation (or an incident
review) reads first.

## 4. THE EXPOSURE PATH (fire NOW, in this order)

```
E1 ROTATE the exposed key immediately (this procedure, fast path: skip nothing,
   but do the create+cut+revoke in one session -- the leaked key stays live only
   for minutes).
E2 SCRUB the leak surface: remove the literal, replace with {file:} reference or
   env var. Commit-history note: if the secret reached a PUBLIC repo or a served
   SKILL.md, treat it as burned FOREVER -- rotation makes it dead, but also scrub
   the history or accept the old key is permanently public (revoked = harmless).
E3 CHECK provider-side abuse: the dashboard's usage log for the exposure window.
   Unexpected usage = incident, not just rotation (log it to the cybersec brain).
E4 Re-run the leak scan (secrets-leak-hunter class) on the affected surface to
   confirm zero remaining literals.
E5 Update the audit log with the exposure entry (what leaked, where, when fixed).
```

Order matters and is non-negotiable: rotate FIRST (the old key is already burned;
scrubbing before rotating just documents the key for longer).

## 5. THE SERVE-PASSWORD SUBPATTERN (highest-frequency rotation in the fleet)

`opencode serve` rotates its password on restart; consumers broke ~10x across the
campaign (EXEC x4, R1B, G1, P2, C3...). The same five phases apply, compressed:

```
S1 After any serve restart, recapture the password from the NEWEST /tmp/oc-serve*.log
   (mtime-sorted, not name-sorted; old logs lie).
S2 Verify: HTTP 200 on the health route + the password length sanity check.
S3 Push the new credential to every downstream consumer THAT IS LIVE NOW (the
   dual-root/multi-consumer law), and mark stale in-context copies as dead.
S4 Consumers acknowledge by completing one authenticated call. Only then is the
   rotation closed. (The R1 lesson: "downstream consumers must re-read it from the
   newest log" -- this skill is that sentence, executable.)
```

## 6. VERIFICATION COMMANDS (paste, never paraphrase)

```bash
# New key works (the only proof that matters):
curl -sS -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $(cat /home/rabi/.keys/<p>-new.key)" https://<provider>/v1/models
# expect: 200

# Old key is dead after revoke:
curl -sS -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $(cat /home/rabi/.keys/<p>-old.bak)" https://<provider>/v1/models
# expect: 401 (record the code; anything else = revocation UNPROVEN)

# No plaintext anywhere in configs/skills after the swap:
grep -rnE "(xpl_|sk-|ar_al)[A-Za-z0-9_-]{10,}" ~/.config/opencode /srv/skills 2>/dev/null
# expect: no output

# Perms on every key file you touched:
stat -c "%a %n" /home/rabi/.keys/*.key        # every row: 600
# Dual-root parity after the swap (per box):
for R in ~/.config/opencode <dotroot>; do python3 -c "import json5;json5.load(open('$R/config.jsonc'))" && echo "$R OK"; done

# Consumer health through the new key:
curl -sS -H "Authorization: Bearer $(cat /home/rabi/.keys/<p>-new.key)" ... (completion probe, provider-failover section 6)
```

## 7. KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---|---|---|
| Keys in a shipped skill file (ai-jury class) | Key pasted inline into SKILL.md; no storage rule was enforced at authoring time | Rotate IMMEDIATELY (the file is already public); externalize to env/{file:}; add the literal-scan gate to the authoring flow |
| One box authenticated, another 401s after rotation | Dual-root drift (one root missed) | dual-root law: ALL roots ALL boxes; round-trip parse + parity count per root |
| Rotation "done" but a seat dies an hour later | Ack was config-diff, not a live authenticated call | Phase C ack = one real completion per consumer, recorded |
| Old key revoked too early; consumers break mid-run | Revoke-before-ack | The inversion is the skill: ack FIRST, revoke SECOND, grace between |
| Same key found in a THIRD location a week later | Inventory was per-config, not fleet-wide (logs, transcripts, backups, served files) | Re-run the section-2 scans including /tmp, evidence dirs, and the served catalog tree; add missed classes to the map |
| New key works in curl, fails in the app | Consumer caches the old secret / stale env | Restart/re-read through the consumer + probe THROUGH it (provider-failover 4.5 pattern) |
| Provider rejects multi-key window | Some providers allow exactly one active key | Shrink grace to the cutover beat; do C ordering tight: cut consumers in one beat, revoke immediately after all acks |
| Rotation reverted silently by a sync | The old config re-synced from a stale root after the swap | Prune/parity (sync-prune) + re-verify post-sync; rotation and sync both write to roots, so they must both follow the dual-root law |

## 8. STORAGE LAW (how keys may exist, fleet-wide)

```
1. Keys live in files with mode 0600 (Windows: ACL to the service user only),
   referenced as {file: ...} in configs -- NEVER inline literals in configs,
   skills, agents, logs, or chat.
2. No key material in: SKILL.md, agent definitions, CONTINUE-HERE files, runlogs,
   beats, transcripts, or brain notes. Reference the key-file PATH only.
3. Backups of key files inherit the mode (cp -p / explicit chmod after copy).
4. The audit log stores names + dates + probe codes, NEVER values.
5. Honeytoken (recommended by the 2026 pipeline guidance): one canary credential
   per provider, used by nothing, with an alert if it is ever USED. Its only job
   is to scream when it leaks.
6. Scans (section 2) run: at every rotation, at every skill-authoring/pull-sync
   (wire the scan into the pull-sync hook -- RGAP2's verdict: mechanical checks
   belong in hooks, deterministic and free), and before every public publish.
```

## 9. RELATION TO EXISTING SKILLS (dedup)

- secrets-leak-hunter / secrets-scanning: DETECT leaks (pre-commit, repo scans).
  This skill is the RESPONSE: rotate + handoff + revoke + audit. They chain; neither
  replaces the other. Detection without rotation leaves the leaked key live.
- dual-root-config-law candidate (RGAP1 #2): this skill inherits its per-root
  verification steps for the secret-bearing fields; the candidate skill remains the
  general config-change procedure, this skill the credential-specific one.
- paperclip-ledger-guard / anti-overclaim: the ack + audit log entries here are the
  evidence bundle those skills weigh; a rotation without the log is an overclaim.
- mcp-oauth-link-rule (memory): OAuth/localhost credentials propagate per machine
  after a manual link; API-key rotation (this skill) is fully scriptable and never
  requires the operator -- different credential classes, do not conflate.

## 10. EVIDENCE REQUIREMENTS (per rotation)

A rotation is "done" only with ALL of:

1. The key map (section 2) as it stood at rotation time, with scan outputs.
2. Per-consumer ack lines (real authenticated calls, status + timestamp).
3. The revocation proof (old key -> 401, pasted).
4. The audit-log lines (section E format).
5. The gates line (section 11).

"Rotated in the dashboard" without acks and revocation proof is a key DUPLICATION,
not a rotation; the old key is still live somewhere and the exposure class is
unchanged.

## 11. GATES LINE USAGE

The canonical 14-field line, values never containing "|", not-fired = "skipped: why":

```
gates -- godmode:activation: fired | promptgod: skipped: procedure turn | subagents: 0 seats | honestscore: <N.N> | ideas: fired | deep-thinker: skipped: operational rotation, no research claim | dim-correctness: <N.N> | dim-completeness: <N.N> | dim-robustness: <N.N (grace/revoke order)> | dim-security: <N.N (perms, no literals)> | dim-performance: <n/a: config op, not a perf task> | dim-concurrency: <N.N (per-consumer cutover order)> | dim-docs: <N.N> | brain: <files written>
```

Minimum to close a rotation turn: key map, ack evidence per consumer, revocation
proof (401 on old), audit-log line count > 0, zero literals in the final scan.

## 12. SELF-TEST (~15 min, uses one throwaway key)

```
12.1 Seed a fake provider entry with an inline literal in a TEST config. Run the
     section-2 inline scan. EXPECT: the literal is found. (If the scan misses its
     own seed, the gate is inert -- the attack-your-own-gate rule.)
12.2 Rotate it per the five phases on ONE test root. EXPECT: new key probes 200,
     old key revoked and probes 401, audit log has >= 5 lines, no downtime window
     for the canary consumer.
12.3 Simulate the dual-root trap: update one root only. EXPECT: the parity check
     (section 6) FAILS and blocks the "done" claim.
12.4 Rotate back (restore old as new) to leave the box in its original state;
     verify with a completion probe and hash the restored key file.
```

A rotation procedure that has never been exercised on a real key leaves the fleet
exactly where it was: rotation-on-human-memory, with the ai-jury class one paste
away from recurring.

## 13. WORKED EXAMPLE (the ai-jury class, run to completion)

```
Trigger: audit found EL_KEY=xpl_42e8... and AR_KEY=ar_al3a... inline in a PUBLIC
SKILL.md (skill-audit-batch1 finding).
E1  Rotate FIRST: created el-2026-11-12-r2 + anonrouter-2026-11-12-r2 in dashboards;
    wrote both to /home/rabi/.keys/ (0600); probes: models 200 + completion ok.
C   Cutover: patched skill-ai-jury to read EL_KEY/AR_KEY from env/{file:}; the jury
    seats re-verified with one real scoring round (the ack).
D   Revoked the leaked keys; old-key probe = 401 on both. Recorded in audit log.
S   Scrubbed the SKILL.md literals, grep clean, catalog rebuilt, sha re-published.
What NOT to claim: "fixed" before revocation proof, or while the old keys were
still live in any dashboard. The rotation is the revocation; everything else is
prep.
```

## 14. VERSION / CHANGELOG

```
secret-rotation v1.0 (2026-11-12)
- Authored from skill-audit-batch1 (ai-jury plaintext-key class: live keys in a
  publicly fetchable SKILL.md), runlog-RGAP1 (serve-password rotation recurrence,
  dual-root law), runlog-RGAP2 item 3 (GitGuardian: scheduled rotation, never
  human memory; agents leak at elevated rates; honeytokens).
- Five-phase procedure: prepare -> deploy-dual -> cutover+ack -> revoke-after-ack
  -> audit. Storage law section 8 is the fleet-wide invariant the gates line checks.
```

## 15. CADENCE AUTOMATION (so rotation never depends on memory)

```
15.1 Schedule (cron or systemd --user timer, or the goal scheduler):
     first day of every 3rd month, 04:00 -> rotate-cadence.sh
15.2 rotate-cadence.sh does:
     - refresh the section-2 inventory scans (new keys since last map update)
     - rotate keys by age (oldest first, N per beat)
     - append a cadence line to the audit log EVEN IF nothing rotated (a beat
       with no side-effect is still written -- silence is not evidence of health)
15.3 Exposure trigger (hook-shaped, zero context cost): the inline-literal scan
     (section 2) added to the skill-authoring and pull-sync gates so a literal can
     never re-enter a shipped file unnoticed.
15.4 Blocked rotation (provider down, multi-key unsupported, cutover needs a
     maintenance window): record ROTATION-BLOCKED in the audit log with the
     reason and retry policy. Blocked-but-logged is an honest state; silently
     deferring is the human-memory failure the cadence exists to kill.
```

## 16. KEY MAP (the living inventory -- section 2's output, maintained)

```markdown
# KEY MAP -- <box> (updated <date>)
| secret | provider | stored at | referenced by | last rotated | next due |
|---|---|---|---|---|---|
| el | example-llm | ~/.keys/el.key (0600) | vps1:~/.config/opencode + dotroot, jury env | 2026-11-12 | 2027-02-10 |
| ar | anonrouter | ~/.keys/ar.key | vps2: BOTH roots, juror C/D/E pins | 2026-11-12 | 2027-02-10 |
| serve-pw | opencode serve | NEWEST /tmp/oc-serve*.log | downstream consumers recapture | per restart | n/a: ephemeral |
| cortex-* | cortex/gateai | per cortex-router config | router primary+fallback | <date> | <date> |
```

Map rules:
- One row per SECRET (not per copy); "referenced by" is the list phase C walks.
  A secret missing from the map is, by definition, unrotatable safely.
- Dual-root boxes list BOTH roots per reference, or the map lies.
- Reviewed at every rotation and every new-provider onboarding; rows older than
  their cadence show in the beat as OVERDUE, not as "probably fine".

## 17. THE OWNER-TOUCH BOUNDARY

Creating a credential for a NEW provider is owner-touch class (a) -- only he can
supply it. Everything else (storage, mapping, dual-root deploy, probes, revocation,
audit) is agent work. A rotation returns AT MOST one owner item ("paste the new
<provider> key") and never a checklist; if the key already exists in the fleet,
rotation is fully autonomous end-to-end.

## 18. GLOSSARY

- key map: the living inventory (section 16) -- one row per secret, referenced-by
  list, rotation dates.
- grace window: the period where old and new keys are both valid; consumers cut
  over during it; the old key dies at the end of it, never before all acks.
- ack: one real authenticated call per consumer with the NEW key. Config looks
  correct is not an ack.
- revoke-after-ack: the invariant that the old key dies only after every consumer
  has proven itself on the new one.
- exposure: any appearance of key material outside its storage location (repo,
  transcript, log, served file, chat). Exposure fires rotation FIRST, scrub second.
- OVERDUE: a map row past its cadence date; treated as an open finding, not a note.

## 19. PROVIDER CAPABILITY CLASSES (how the five phases adapt)

Not every provider supports the same rotation mechanics. Classify first:

| Class | Provider behavior | Procedure adaptation |
|---|---|---|
| A: multi-key native | old + new valid simultaneously, self-serve revoke | Full zero-downtime five phases as written (the default) |
| B: one-active-key | creating a key invalidates the old (or cut is instant) | Phases B/C collapse into one beat: cut ALL consumers in one ordered pass (canary -> long seats -> batch), revoke is implicit; the ack list is written BEFORE the cut, and any consumer that fails its ack is restored from the .pre-swap backup, not debugged live |
| C: one-key + delayed revoke | new key usable, old key revocable only later | Enforce the ack deadline; if a consumer has not acked by the revoke deadline, it stays on the map as BLOCKED and the revoke is deferred with a recorded reason -- never revoke with unacked consumers |
| D: dashboard-only (manual portal cut) | key must be swapped by hand in a web UI | Prepare everything agent-side (map, staged configs, probe scripts), then hand the operator exactly ONE item (the portal action); probes + revocation proof run the moment the cut lands |

Misclassifying B as A is how mid-run breaks happen: the "grace window" never
existed, and the first consumer cut over while three others were mid-call.

## 20. FAILURE MODES DURING ROTATION (and the recovery for each)

| Failure at phase | Consequence | Recovery |
|---|---|---|
| A: new key invalid (typo, wrong scope) | Probes fail before any cutover | Nothing broke; delete the bad key, fix, re-prepare. No consumer ever saw it |
| B: cutover leaves one consumer on old key | That consumer dies at revoke time | Extend grace; ack it; only then revoke. The map's ack column is the tracker |
| C: revoke happens with an unacked consumer | Outage on that consumer | Roll back the consumer to its .pre-swap config (old key may be dead -- if so, reissue from the dashboard; this is why B/C need the pre-swap backup too) |
| D: portal cut done but probe fails | New key bad, old key possibly dead | Class D cut is one-way: restore = reissue from portal; log as a rotation incident, add the provider class to the map |
| Any: key material echoed in a transcript/beat | Re-exposure of the NEW key | Treat as a fresh exposure: rotate again (the leaked key must die), fix the printing code, scrub the artifact |

## 21. INTEGRATION MAP (which skills hand off to this one)

- provider-failover: its config-plane swap (its section 4) embeds a key write; a
  swap that introduces a NEW key triggers this skill's phases A (inventory) and B
  (deploy) inline. Breaker states are NOT secrets and do not rotate.
- sync-prune: pulled skills enter the fleet through the sync; the pull gate wires
  the section-2 literal scan so no incoming SKILL.md ships keys (the ai-jury class
  enters exactly this way -- a published file with inline secrets).
- dual-root-config-law: the deployment mechanics of phase B; that candidate skill
  (config law) and this skill share the parity-check commands verbatim.
- secrets-leak-hunter / secrets-scanning: detection layer, fires before this skill;
  rotation is the response they cannot perform themselves.
- unicode-tags-sanitiser / mcp-auditor: adjacent authoring gates; neither rotates
  anything, both benefit from the storage law holding fleet-wide.

## 22. PRE-ROTATION CHECKLIST (the run's opening state; all boxes ticked or explained)

```
[ ] Key map exists for this box and has a row for the target secret
[ ] Every referenced-by root enumerated (both roots on dual-root boxes)
[ ] Provider rotation class determined (A/B/C/D, section 19)
[ ] .pre-swap backups of every file to be touched, hashed
[ ] New key created, stored 0600, probe passed (models 200 + completion ok)
[ ] Shared/grace windows and the ack list written down before the cutover
[ ] Owner-touch item identified (at most one: new provider key) or none needed
```

## 23. POST-ROTATION CHECKLIST (evidence bundle; each line cites a command output)

```
[ ] Every consumer acked on the NEW key (one real authenticated call each, logged)
[ ] Old key revoked; probe on OLD key returns 401 (pasted)
[ ] Parity: both roots parse, both roots reference the new key file, counts match
[ ] Inline-literal scan clean across configs, skills, /tmp, transcripts
[ ] Audit log (append-only) carries: created, deployed, acked-per-consumer,
    revoked, scrubbed -- names and dates only, zero key material
[ ] Key map updated: last-rotated, next-due, class noted
[ ] Gates line emitted with honestscore N.N and the audit-log path in brain:
```

A rotation closed with prose only ("keys rotated, all good") and no command output
is an overclaim; the checklists exist so the honest path is also the shortest one.

## 24. FREQUENCY GUIDANCE (calibrated, with the honest caveat)

| Secret class | Cadence | Basis |
|---|---|---|
| Provider API keys (crew-facing, high blast radius) | 90 days | The stated default cadence; GitGuardian guidance is "schedule or lease, never memory" |
| Provider API keys (read-only / low blast) | 180 days | Proportionality; note the class in the map |
| Serve/runtime passwords | Per restart (already the fleet pattern) | Ephemeral by design; consumers recapture from the newest log |
| OAuth/refresh tokens | On the provider's lifecycle + any suspected exposure | Not fully scriptable; the map tracks the dates either way |
| SSH keys / deploy keys | 12 months or on role change | Out of the LLM-key path but lives in the same map |

Honest caveat: 90 days is the adopted default, not an externally mandated constant --
the defensible rule is "a cadence that exists and is machine-tracked beats any
particular number." The map's OVERDUE flag is what makes the cadence real; without
it, the number is decoration.

## 25. MINIMUM VIABLE ROTATION (one box, one key, class A -- the whole loop in 10 commands)

```bash
# 1. inventory: where does the old key live?
grep -rn "xpl_" ~/.config/opencode /srv/skills 2>/dev/null
# 2. new key in, stored safely:
umask 077 && printf '%s' "$NEW_KEY" > /home/rabi/.keys/el-2026-11-12.key
# 3. prove NEW:
curl -sS -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $(cat /home/rabi/.keys/el-2026-11-12.key)" https://api.example.com/v1/models
# 4. cutover both roots + consumer, ack per consumer:
#    (config patch -> restart -> one real authenticated call each; record statuses)
# 5. revoke OLD in the dashboard; prove dead:
curl -sS -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $(cat /home/rabi/.keys/el-old.bak)" https://api.example.com/v1/models
# 6. scrub + rescan:
grep -rn "xpl_" ~/.config/opencode /srv/skills /tmp/*.log 2>/dev/null   # expect: no hits
# 7. audit + map update (names and dates only -- never the key itself)
```

Ten commands, each with a pasteable output. That is the entire rotation on the
simplest possible shape; every larger case (multi-box, multi-consumer, class B/C/D)
is this loop with more consumers in step 4 and more rows in the map.
