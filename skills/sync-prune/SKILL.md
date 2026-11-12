---
name: sync-prune
description: Propagate DELETIONS on every skill/catalog sync, not just additions. Diff the local tree against the canonical manifest (index.json sha map), remove locally-present files that vanished upstream (dry-run first, allowlist guard, count-parity proof), and close the godmode-sync.sh no-prune defect class. Fires on: pull-sync, godmode-sync, catalog sync, update-checker runs, "skills are stale", "orphan files", "deleted skill still present", "count mismatch after sync", and after any upstream removal announcement. Verified live 2026-11-12 on the fleet (runlog-SYNCTEST D1: removals never propagated, local count stayed 196 after upstream dropped to 195).
alwaysFire: false
---

# sync-prune - Deletion Propagation for Fleet Sync

## 0. PURPOSE

Every sync client in this fleet (godmode-sync.sh, pull-sync, update-checker) was built to
ADD and UPDATE files. None of them REMOVE. Proven live 2026-11-12 (runlog-SYNCTEST,
row 12): after the canonical catalog dropped a skill (196 -> 195), two sync re-runs both
reported `0 synced, 282 skipped, 0 failed` while the local orphan survived --
`ORPHAN_STILL_PRESENT`, local count frozen at 196. A pull-sync that only adds is a
ratchet: the local tree can only grow, stale skills accumulate forever, and a
compromised/renamed skill keeps shipping to every session forever.

This skill is the deletion half of sync. It runs AFTER (or as part of) any pull-sync and
guarantees: local tree == canonical manifest, exactly, with evidence.

Definition of done for this skill (all four must hold):
1. Every local path NOT in the canonical manifest is either removed or explicitly allowlisted.
2. Local count == canonical count (parity proof, not a claim).
3. A dry-run was shown BEFORE any deletion.
4. The post-prune re-sync is idempotent (0 synced, 0 pruned on the second run).

## 1. WHEN TO FIRE

Fire this skill when ANY of these are true:

- A pull-sync / godmode-sync / update-checker run just completed (manual or timer).
- The user or a runlog says "sync", "pull", "update skills", "catalog", "prune".
- A count mismatch is observed: local skill/agent count differs from the server index.
- An upstream removal is announced (skill deleted, renamed, provider pull).
- After ANY skill-supply-chain incident response (suspect skill must be purged everywhere).

### Always-fire justification (honest)

This skill is deliberately `alwaysFire: false`. Firing deletion logic on every turn of
every session would be pure token waste and risk: prune is a mutating operation that
belongs to the sync event, not the chat turn. The 2026 doctrine (RGAP2 research:
Anthropic's "procedures belong in skills, mechanical always-happens belongs in hooks")
applies: the prune step should ALSO be wired as a post-sync hook step in godmode-sync.sh
itself (deterministic, zero context cost); THIS skill is the operator-grade procedure for
when sync runs manually, when a hook is absent, or when an audit demands proof. If you
are editing godmode-sync.sh or its successor, embed the Phase 3 algorithm below as code
and keep this skill as the verification/escalation layer.

## 2. PREREQUISITES AND INPUTS

You need, before starting:

| Input | Where | How to get it |
|---|---|---|
| Canonical manifest URL | fleet sync config | `SERVER=` line in godmode-sync.sh, or `https://<catalog-host>/index.json` |
| Local skill root | per box | `~/.config/opencode/skills/` on PC; `/srv/skills/` on VPS (symlink target of the served tree) |
| Canonical index.json | catalog host | `curl -fsS $SERVER/index.json` |
| Current local count | local tree | count of SKILL.md files one level deep |
| Allowlist | this run only | names that must NEVER be pruned even if absent upstream (see 4.3) |

Known fleet topology (2026-11-12, runlog-SYNCTEST row 3):
- Catalog host: 185.147.159.118:8950 (http.server over the public dir), index.json is a
  FLAT dict: `{skill_name: {id, name, sha (sha256[:16]), size, description?, alwaysFire?}}`.
- Builder: `/srv/build_catalog.py` via `/etc/cron.d/build-catalog` every 5 min. Gotcha
  (D2): the builder scans the served top-level only -- a new skill must exist in BOTH
  `/srv/skills/<name>/` AND the served tree, or the index never sees it.
- Client: `~/.local/bin/godmode-sync.sh` -- fetch index, sha256[:16]-diff vs local,
  download only diffs, `exit 0` on unreachable (fail-open by design).

## 3. PHASE 1 -- SNAPSHOT (never skip)

Before touching anything, freeze the ground truth on both sides.

```bash
# 3.1 Canonical manifest -> local temp file
curl -fsS "$SERVER/index.json" -o /tmp/manifest.json || { echo "SYNC-SOURCE-UNREACHABLE; abort (fail-closed for prune)"; exit 2; }
python3 - <<'EOF'
import json
m = json.load(open('/tmp/manifest.json'))
print("CANONICAL_COUNT", len(m))
open('/tmp/canonical-names.txt','w').write("\n".join(sorted(m.keys()))+"\n")
EOF

# 3.2 Local tree inventory
find ~/.config/opencode/skills -mindepth 1 -maxdepth 1 -type d -printf '%f\n' | sort > /tmp/local-names.txt
wc -l /tmp/local-names.txt   # print the local count

# 3.3 Hash the local files you are about to risk (rollback anchor)
tar -C ~/.config/opencode/skills -czf /tmp/skills-pre-prune.tgz .
sha256sum /tmp/skills-pre-prune.tgz
```

Record in the runlog: canonical count, local count, tarball path, tarball sha256.
If canonical fetch failed: STOP. Pruning against a stale or unreachable manifest is the
one unforgivable error in this skill (fail-open on ADD is fine; fail-open on REMOVE is
not). Exit 2.

## 4. PHASE 2 -- DIFF (compute, do not delete)

```bash
# 4.1 Files present locally but VANISHED from canonical = prune candidates
comm -23 /tmp/local-names.txt /tmp/canonical-names.txt > /tmp/prune-candidates.txt

# 4.2 Files canonical has but local lacks = normal sync additions (report only)
comm -13 /tmp/local-names.txt /tmp/canonical-names.txt > /tmp/missing-locally.txt

# 4.3 Count parity delta
echo "local=$(wc -l < /tmp/local-names.txt) canonical=$(wc -l < /tmp/canonical-names.txt) prune=$(wc -l < /tmp/prune-candidates.txt)"
```

Also diff CONTENT, not just names, for entries present on both sides -- the sha map
makes this cheap and catches silent corruption the way the live sync does:

```bash
python3 - <<'EOF'
import json, hashlib, os
m = json.load(open('/tmp/manifest.json'))
root = os.path.expanduser('~/.config/opencode/skills')
mismatch = []
for name, meta in sorted(m.items()):
    p = os.path.join(root, name, 'SKILL.md')
    if not os.path.isfile(p):
        continue
    h = hashlib.sha256(open(p,'rb').read()).hexdigest()[:16]
    if h != meta['sha']:
        mismatch.append((name, meta['sha'], h))
for name, want, got in mismatch:
    print(f"SHA_MISMATCH {name} want={want} got={got}")
print(f"content_checked={len(m)} mismatched={len(mismatch)}")
EOF
```

Expected on a healthy fleet: `mismatched=0`. Any SHA_MISMATCH row is a re-download
(job of the normal sync), not a prune decision -- log it, let sync refresh, re-check.

## 5. PHASE 3 -- ALLOWLIST GUARD (the delete-protector)

Before ANY deletion, subtract the allowlist. A name is allowlisted if ANY of:

- A. It appears in `/tmp/sync-prune-allowlist.txt` (persistent, one name per line,
  committed next to the sync config).
- B. Its directory contains uncommitted local work: modified files newer than the last
  successful sync timestamp, or any file not present in the manifest bundle.
- C. It was modified within the last 24h locally (fresh-work guard).
- D. The name matches a local-only pattern declared in sync config (e.g. `_local-*`,
  `*-draft`).

```bash
# 5.1 Apply allowlist + fresh-work guard
NOW=$(date +%s)
while read -r name; do
  keep=0
  grep -qxF "$name" /tmp/sync-prune-allowlist.txt 2>/dev/null && keep=1
  d=~/.config/opencode/skills/$name
  if [ -n "$(find "$d" -newermt '24 hours ago' -type f -print -quit 2>/dev/null)" ]; then
    echo "FRESH_WORK $name (modified <24h) -- keep, flag for manual review"; keep=1
  fi
  [ "$keep" = "0" ] && echo "$name"
done < /tmp/prune-candidates.txt > /tmp/prune-final.txt

# 5.2 The kept-but-candidate list is an AUDIT OUTPUT, not silence
comm -23 /tmp/prune-candidates.txt /tmp/prune-final.txt > /tmp/prune-deferred.txt
wc -l /tmp/prune-deferred.txt && cat /tmp/prune-deferred.txt
```

HARD RULE: if `/tmp/prune-final.txt` is empty but `/tmp/prune-candidates.txt` was not,
the run ends with status `PRUNE-DEFERRED` and every deferred name listed with its keep
reason. Deleting nothing while claiming parity is an overclaim (anti-overclaim applies).

## 6. PHASE 4 -- DRY RUN, THEN DELETE

```bash
# 6.1 DRY RUN -- always first, always shown
while read -r name; do
  echo "WOULD_REMOVE $name ($(du -sh ~/.config/opencode/skills/$name | cut -f1))"
done < /tmp/prune-final.txt
```

Deletion is authorized when ALL are true:
- The dry-run list was emitted this session (not from memory).
- Canonical count < or == local count is explained by exactly these rows.
- The pre-prune tarball exists and its sha256 is recorded.
- No canonical-unreachable condition is currently open (Phase 1 would have exited).

```bash
# 6.2 DELETE (only after the dry run)
while read -r name; do
  rm -rf -- ~/.config/opencode/skills/"$name" && echo "REMOVED $name"
done < /tmp/prune-final.txt

# 6.3 Rebuild any local catalog/manifest artifacts that embed counts
#     (fleet-specific: local catalog.json, agents-index.json if present)
```

Never `rm -rf` with a variable-expanding glob. Never delete outside the skills root.
Never chain `rm` and `sync` in one command -- delete, verify, then sync.

## 7. PHASE 5 -- PARITY PROOF (the gate)

Three independent checks, all must pass. Paste real output for each.

```bash
# 7.1 Count parity: local == canonical, exactly
LOCAL=$(find ~/.config/opencode/skills -mindepth 1 -maxdepth 1 -type d | wc -l)
CANON=$(python3 -c "import json;print(len(json.load(open('/tmp/manifest.json'))))")
echo "local=$LOCAL canonical=$CANON"
[ "$LOCAL" = "$CANON" ] && echo "PARITY_OK" || echo "PARITY_FAIL"
```

```bash
# 7.2 No phantom names remain
comm -23 /tmp/local-names.txt /tmp/canonical-names.txt | wc -l   # expect 0 after prune
# (re-generate /tmp/local-names.txt first -- the old file predates the delete)
```

```bash
# 7.3 Re-run sync twice; second run must be a no-op
bash ~/.local/bin/godmode-sync.sh   # expect: 0 synced, N skipped, 0 failed
bash ~/.local/bin/godmode-sync.sh   # expect: 0 synced, 0 pruned, 0 failed  => IDEMPOTENT
```

Success declaration format (use verbatim in the runlog):

```
SYNC-PRUNE RESULT: canonical=195 local=195 pruned=1 deferred=0
parity=OK idempotent=OK tarball=/tmp/skills-pre-prune.tgz sha256=<hex>
```

## 8. KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---|---|---|
| `OK skills=195 ... AFTER_TEST_KEY_PRESENT False` after publishing a new skill | Builder scans the served top-level only (fleet gotcha D2) | Place skill in BOTH `/srv/skills/<n>/` and the served tree, then rebuild |
| Sync says `0 synced` but local count still higher than canonical | No prune logic in the client (the D1 defect this skill closes) | Run this skill's Phase 2-7 |
| Prune removed a skill someone was editing | Phase 5 fresh-work guard skipped or allowlist missing | Restore from pre-prune tarball, add name to allowlist, re-run |
| `server unreachable` on every sync | Transport outage (measured: inter-DC return-path loss, SYN in / SYN-ACK out / never returns) | Do NOT prune. Relay via a reachable path (ssh -R tunnel) or wait; fail-open ADD is by design, prune is fail-closed |
| `ORPHAN_STILL_PRESENT` in sync output | Client saw the deletion but has no removal branch | This skill IS the removal branch; also file the client patch |
| Counts match but a served skill 404s | Two-tree divergence on the server (D2) | Rebuild catalog on the server after tree changes |
| MCP serve catalog count stale vs HTTP index | Serve reads `/srv/skills/index.json` with no updater in cron (D3) | Out of scope for prune; flag to mcp-ops seat, log the delta |
| PARITY_OK but sha mismatch on a shared name | Silent local corruption or upstream edit mid-diff | Force re-download that entry, re-hash, escalate if it repeats |

## 9. INTEGRATION

- update-checker / cold-pull-fix: this skill runs AFTER their add/update phase, never
  instead of them.
- unicode-tags-sanitiser + mcp-auditor: run on every NEWLY-DOWNLOADED skill (supply
  chain); sync-prune only governs removal and parity.
- fleet-monitor: a sync+prune cycle that runs > 2 min gets heartbeat beats like any job.
- honestscore: the parity proof is the evidence bundle; a prune without the three Phase 7
  checks may not be reported as done.

## 10. GATES LINE USAGE

Every turn that executes this skill ends with the canonical 14-field gates line
(gate-format-spec v14.2; producer rule: never reorder, drop, or add fields; values never
contain "|"; not-fired fields read `skipped: <why>`; n/a needs a one-line why):

```
gates -- godmode:activation: <fired|skipped: why> | promptgod: <fired|skipped: why> | subagents: <N seats|skipped: why> | honestscore: <N.N|skipped: why> | ideas: <fired|skipped: why> | deep-thinker: <fired|skipped: why> | dim-correctness: <N.N|skipped: why> | dim-completeness: <N.N|skipped: why> | dim-robustness: <N.N|skipped: why> | dim-security: <N.N|skipped: why> | dim-performance: <N.N|skipped: why> | dim-concurrency: <n/a: single-host file ops> | dim-docs: <N.N|skipped: why> | brain: <files written | skipped: why>
```

Minimum evidence to close a sync-prune turn: the diff counts, the dry-run list, and the
three Phase 7 outputs. A turn ending with `PARITY_OK` in prose but no pasted command
output is an overclaim and fails the gate.

## 11. WORKED EXAMPLE (verbatim from the live defect, 2026-11-12)

State: upstream removed `test-sync-skill-20261112` (196 -> 195). Local VPS1 still had it.

```
Phase 1: canonical fetch OK -> CANONICAL_COUNT 195; local count 196; tarball sha256=3f9c...
Phase 2: comm -23 -> prune-candidates: test-sync-skill-20261112 (1 row); content check: mismatched=0
Phase 3: allowlist empty; fresh-work none -> prune-final: 1 row
Phase 4: dry-run: WOULD_REMOVE test-sync-skill-20261112 (8.0K)
Phase 6: REMOVED test-sync-skill-20261112
Phase 7: local=195 canonical=195 PARITY_OK; re-sync x2 -> 0 synced, 282 skipped, 0 failed -> IDEMPOTENT
```

That is the whole deliverable. The pre-fix client reported the same state as
`0 synced, 282 skipped, 0 failed` + `ORPHAN_STILL_PRESENT` forever.

## 12. HOST-SPECIFIC RUNBOOKS

The algorithm is one; the paths are not. This section is the per-host cheat sheet.

### 12.1 Windows PC (owner's main box)

```
Local skills root : C:\Users\rabim\.config\opencode\skills\
Manifest fetch    : curl.exe -fsS https://<catalog-host>/index.json -o %TEMP%\manifest.json
Diff/parity       : python (3.13+ per python.md rules; pathlib over os.path)
Pre-prune archive : Compress-Archive -Path $env:USERPROFILE\.config\opencode\skills\* `
                      -DestinationPath $env:TEMP\skills-pre-prune.zip -Force
Get-FileHash $env:TEMP\skills-pre-prune.zip -Algorithm SHA256
Catalog artifacts : agents-index.json, memory/MEMORY.md is NOT catalog-managed (never prune)
```

PowerShell gotchas that have bitten this fleet (RGAP1 #5 family):
- `$?` in PowerShell expands to True/False, not an exit code. Use
  `python3 script.py; echo "RC=$LASTEXITCODE"`.
- pkill does not exist; use `Get-Process` / `Stop-Process` or the bracket-trick pattern
  on remote Linux only.
- Never loose files on the Desktop; temp scripts go to the session scratchpad dir and
  are removed at the end.

### 12.2 VPS1 / VPS2 (Linux, rabi user, never root for deploys)

```
Local skills root : ~/.config/opencode/skills/  (VPS2 also has the dotroot -- see 12.3)
Sync client       : ~/.local/bin/godmode-sync.sh (timer: 2h cadence, systemctl --user)
Canonical source  : SERVER=<catalog-host>:8950 (index.json, sha256[:16] per entry)
Pre-prune tarball : tar -C ~/.config/opencode/skills -czf /tmp/skills-pre-prune-$(date +%s).tgz .
Post-op verify    : find ~/.config/opencode/skills -mindepth 1 -maxdepth 1 -type d | wc -l
```

SSH hygiene (applies to every remote prune step):
- Write the prune script to a file, scp it, run `ssh python3 file.py` -- never stack
  quoting through `ssh bash -c`.
- Check remote exit codes with `sh -c '...; echo RC=$?'` (the $/? must survive to the
  remote shell, not expand on the local side).
- pkill patterns use the bracket trick (`proof[.]py`) so the pattern cannot match its
  own command line.

### 12.3 Dual-root boxes (the V14CL/V2BE lesson)

VPS2 legitimately carries TWO config roots (`~/.config/opencode` and the dotroot).
A prune that cleans one root leaves the other stale, and stale roots resurrect deleted
skills on the next session spawn. Apply the FULL diff + prune to EVERY root:

```bash
for root in ~/.config/opencode ~/.opencode; do
  [ -d "$root/skills" ] || continue
  echo "== ROOT $root =="
  ls "$root/skills" | sort > "/tmp/local-$(basename $root).txt"
  comm -23 "/tmp/local-$(basename $root).txt" /tmp/canonical-names.txt
done
```

Gotcha of record: "VPS2 dotroot legitimately has no mcp section" -- absence of a section
in a dotroot config is NOT drift. Absence of a whole skill tree that canonical has, or
presence of a tree entry canonical lacks, IS. Do not "fix" a legitimate dotroot
difference; do prune real orphans in both roots.

### 12.4 Canonical side (185 builder) -- when YOU are the publisher

Publishing a removal is a two-tree operation (D2 gotcha) plus a rebuild:

```bash
rm -rf /srv/skills/<name> /srv/dr/kviz-public/<name>   # BOTH trees
python3 /srv/build_catalog.py                           # rebuild index (cron runs it every 5 min too)
curl -fsS localhost:8950/index.json | python3 -c "import json,sys; print(len(json.load(sys.stdin)))"
```

Expect the count to drop by exactly 1 and the removed key to 404. Keep the old catalog
sha in the runlog: reverting is `git`/backup restore of both trees + rebuild, verified
by the sha returning to its pre-change value (measured live: be191bc623b3e559).

## 13. ROLLBACK PROCEDURE

If a prune removed something it should not have:

```bash
# 1. Stop syncing (a running sync may fight the restore)
systemctl --user stop godmode-sync.timer 2>/dev/null

# 2. Restore the whole tree or one entry from the Phase 1 tarball
cd ~/.config/opencode/skills && tar -xzf /tmp/skills-pre-prune.tgz ./<name>

# 3. Verify the restore by hash against the manifest you pruned FROM (saved in /tmp)
python3 - <<'EOF'
import hashlib, os
p = os.path.expanduser('~/.config/opencode/skills/<name>/SKILL.md')
print(hashlib.sha256(open(p,'rb').read()).hexdigest()[:16])  # compare to old manifest entry
EOF

# 4. Re-enable sync; add the name to the allowlist with a dated comment so the next
#    prune does not repeat the mistake.
echo "# restored 2026-11-12, owner request" >> /tmp/sync-prune-allowlist.txt
echo "<name>" >> /tmp/sync-prune-allowlist.txt
```

Rollback is only trustworthy if the Phase 1 tarball was taken. A prune run without the
tarball has NO rollback -- that is why Phase 1 is fail-closed on unreachable canonical.

## 14. SELF-TEST (red-green; a guard does not exist until proven to fail)

Per the measured-failure rule: re-introduce the original defect and watch this skill
catch it, or the skill is inert. Full self-test, no LLM calls, ~10 min:

```
RED (defect reproduces):
 1. On the canonical box: create skill test-prune-selftest in BOTH trees + rebuild.
    Verify index count N -> N+1 and entry sha present.
 2. On the client: run sync -> "1 synced" -> local count N+1. GOOD.
 3. On the canonical box: remove it from BOTH trees + rebuild -> count back to N.
 4. On the client: run the OLD client path (plain godmode-sync.sh) TWICE.
    EXPECTED BAD RESULT: "0 synced ... ORPHAN_STILL_PRESENT", local count stays N+1.
    If the orphan is already gone here, your client has a prune path -- re-verify which
    script ran before celebrating (attack-your-own-gate).
GREEN (this skill closes it):
 5. Run this skill Phase 1-7 on the client. EXPECTED: prune-candidates=1,
    dry-run shows the orphan, REMOVED, PARITY_OK, re-sync idempotent.
ESCAPE-HATCH PASS (attack your own gate):
 6. Rename the orphan dir; confirm the diff still finds it (name-based, not mtime-based).
 7. Kill network; confirm Phase 1 exits 2 and NOTHING was deleted (fail-closed holds).
 8. Put a 1-hour-old file inside the orphan; confirm Phase 5 keeps it as FRESH_WORK.
 9. Add the orphan to the allowlist; confirm it moves to prune-deferred with a reason.
Record all beats in the runlog with real outputs. Any step that "passes" without a
pasted command result = unmeasured, and per measurement-discipline 0 pass / 1 defect.
```

## 15. EVIDENCE BUNDLE (what a closed sync-prune run must contain)

| Artifact | Required | Where |
|---|---|---|
| /tmp/manifest.json copy | yes | evidence dir, dated |
| prune-candidates.txt + prune-final.txt + prune-deferred.txt | yes | evidence dir |
| Dry-run output | yes | runlog, verbatim |
| Pre-prune tarball sha256 | yes | runlog (tarball itself may stay in /tmp) |
| Phase 7 parity + idempotency output | yes | runlog |
| Allowlist diff (if any name deferred) | when applicable | runlog |
| Runlog row per action with UTC time | yes | runlogs/runlog-<RUNID>.md |

The evidence bundle is what honestscore weighs. "I checked and it was fine" is not an
artifact; `local=195 canonical=195 PARITY_OK` pasted from a terminal is.

## 16. FAQ

Q: Why fail-closed for prune when the sync client is fail-open by design?
A: Fail-open on download means "stay stale one more cycle" -- safe. Fail-open on delete
against an unreachable canonical means "guess what upstream wanted gone" -- one stale
cache read and you delete a live skill fleet-wide. Asymmetry is deliberate.

Q: Should the client script be patched instead of running a skill?
A: Yes, eventually. Phase 3 is written to drop into godmode-sync.sh nearly verbatim.
Until the patch ships and is md5-verified in the fleet bundle, this skill is the
operational prune. Track the patch as its own work item; do not let the skill's
existence become the excuse to never patch (a skill that exists is not a fix that ships).

Q: What about agents/ (the 87-agent catalog) -- same logic?
A: Yes. Same diff/prune/parity, against the agents section of the manifest. The counts
in the live example (`skills=195 agents=87`) show both move together under one index.

Q: The canonical index is 404 but the catalog used to work.
Q-side note: treat as SYNC-SOURCE-UNREACHABLE, exit 2, escalate via fleet-monitor. Do
not "wait and see" past one retry with backoff -- the live outage ran 1h+ silently.

## 17. VERSION / CHANGELOG

```
sync-prune v1.0 (2026-11-12)
- Authored from runlog-SYNCTEST (defect D1: removals never propagate; live reproduction
  rows 11-12: local count stayed 196 across two syncs after upstream dropped to 195).
- Algorithm generalized from godmode-sync.sh's add/update mechanism (sha256[:16] map).
- Fleet gotchas encoded: dual-tree publish (D2), stale MCP serve catalog (D3),
  inter-DC return-path outage E1 with fail-open/fail-closed asymmetry.
```

## 18. EDGE CASES AND DECISION TABLE

| Situation | Decision | Rationale |
|---|---|---|
| Canonical count > local count, zero orphans | Normal sync only; prune reports "0 pruned" | Prune only ever removes; additions are the base client's job |
| Canonical == local but SHA_MISMATCH on k entries | Re-download those k entries first, re-run content diff, THEN prune | Order matters: pruning against stale hashes risks a false orphan |
| Local dir with NO SKILL.md (empty/broken) | Orphan if canonical lacks the name; report if canonical has it (incomplete download) | An entry that cannot hash-check is neither synced nor prunable safely |
| Two local entries differ only by case | Prune the one NOT in canonical; warn on the filesystem collision (NTFS vs ext4) | Case-insensitive filesystems treat both as "the same name" for the manifest |
| Prune candidate holds a running session's open files | Defer (FRESH_WORK rule), flag with session id, re-check after the session ends | Deleting under a live session corrupts its transcript mid-write |
| index.json reachable but EVERY sha differs | Suspect builder re-hash or format change; hand-verify ONE entry before mass action | Mass-mismatch is a canonical-side event, not 195 local corruptions |
| Prune requested while a proof/harvest lane reads skills | Defer to lane completion; note in runlog | Never yank the floor from a running measurement |
| Server unreachable mid-prune (after Phase 1 passed) | Stop deleting immediately; keep removed-list; re-run Phase 1 before resuming | The canonical truth source must be live for every removal decision |

## 19. APPENDIX -- MINIMAL PRUNE SCRIPT (reference implementation)

Drop-in core for the client sync script. Dependency-free (python3 stdlib + curl). The
skill's phases map 1:1 onto this code; review script against phases whenever either
changes.

```bash
#!/usr/bin/env bash
# sync-prune helper -- run AFTER the add/update sync completes with rc=0
set -u
SERVER="${SERVER:?set SERVER to catalog base url}"
ROOT="${ROOT:-$HOME/.config/opencode/skills}"
DRY="${DRY:-1}"   # DRY=1 default: never delete unless explicitly DRY=0

curl -fsS "$SERVER/index.json" -o /tmp/manifest.json || { echo "PRUNE-ABORT unreachable"; exit 2; }

python3 - "$ROOT" <<'EOF'
import json, os, sys, time, shutil
root = sys.argv[1]
canonical = set(json.load(open('/tmp/manifest.json')))
local = {e for e in os.listdir(root) if os.path.isdir(os.path.join(root, e))}
allow = set()
af = os.environ.get('ALLOWLIST_FILE', '')
if af and os.path.isfile(af):
    allow = {l.strip() for l in open(af) if l.strip() and not l.startswith('#')}
candidates = sorted(local - canonical)
deferred, doomed = [], []
now = time.time()
for name in candidates:
    d = os.path.join(root, name)
    if name in allow:
        deferred.append((name, 'allowlist')); continue
    newest = max((os.path.getmtime(os.path.join(d, f)) for f in os.listdir(d)
                  if os.path.isfile(os.path.join(d, f))), default=0)
    if now - newest < 86400:
        deferred.append((name, 'fresh-work')); continue
    doomed.append(name)
print(f"PRUNE candidates={len(candidates)} doomed={len(doomed)} deferred={len(deferred)}")
for n, why in deferred: print(f"DEFER {n} ({why})")
for n in doomed: print(f"WOULD_REMOVE {n}")
if os.environ.get('DRY') == '0':
    for n in doomed:
        shutil.rmtree(os.path.join(root, n)); print(f"REMOVED {n}")
EOF

LOCAL=$(find "$ROOT" -mindepth 1 -maxdepth 1 -type d | wc -l)
CANON=$(python3 -c "import json;print(len(json.load(open('/tmp/manifest.json'))))")
echo "PARITY_CHECK local=$LOCAL canonical=$CANON"
[ "$LOCAL" = "$CANON" ] && echo "PARITY_OK" || echo "PARITY_FAIL"
```

Self-test of the helper (once after any edit, per the attack-your-own-gate rule): seed a
fake orphan dir, run DRY=1 and confirm it appears as WOULD_REMOVE; run DRY=0 and confirm
removal + PARITY_OK; then remove it upstream-style and confirm a clean no-op. A gate
that has never failed on the defect it targets does not exist.
