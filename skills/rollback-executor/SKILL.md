# rollback-executor

**Executes rollbacks when something goes wrong — catalog index, agent files, tools, stamps. Always restores to last known good state.**

## Triggers

- "rollback this change"
- "restore to previous state"
- "undo that deploy"
- "the catalog is broken, fix it"
- "restore agent to previous version"
- "index.json is corrupt"
- After any failed publish, deploy, or tool add
- When a deploy breaks a downstream consumer
- Any `500` or `404` from a service that was working before your change

---

## Context

Every deploy and every index rebuild in this fleet creates a `.bak` file before overwriting. This skill's job is to find that `.bak`, verify it is valid, restore it atomically, and confirm the restore worked. It covers four rollback surfaces: the Cortex catalog (`index.json`), individual agent files (`agent.md`), vault tool directories, and stamps/version markers. Never manually edit a broken file to fix it — restore the `.bak` first, then make targeted fixes on the clean base.

**Rule**: a rollback is only "done" when you have verified the restored artifact is valid AND the downstream service has recovered (a live request returns the expected response, or the loader reads it without error).

---

## Rollback Surface 1 — Catalog (index.json)

### When to use
- `index.json` returns invalid JSON
- Catalog query returns 500 or missing entries
- A bad rebuild removed legitimate tools/agents
- The file is 0 bytes or truncated

### Locate the backup

```bash
# Cortex/catalog on web-dev VPS
ls -la /srv/dr/kviz-public/catalog/index.json*
# Expected:
#   index.json        ← current (may be broken)
#   index.json.bak    ← last known good

# Tools index
ls -la /srv/dr/kviz-public/tools/tools-index.json*
```

### Restore

```bash
TARGET_HOST="rabi@179.61.138.8"
CATALOG_PATH="/srv/dr/kviz-public/catalog"

ssh $TARGET_HOST "
  set -e
  # Verify the backup exists and is valid JSON
  python3 -m json.tool $CATALOG_PATH/index.json.bak > /dev/null \
    && echo 'BAK valid' \
    || { echo 'ERROR: .bak is also invalid — escalate'; exit 1; }

  # Atomic restore: copy .bak to a temp file, then mv (mv is atomic on same FS)
  cp $CATALOG_PATH/index.json.bak $CATALOG_PATH/index.json.tmp
  mv $CATALOG_PATH/index.json.tmp $CATALOG_PATH/index.json
  echo 'Restored index.json from .bak'

  # Verify restore
  python3 -m json.tool $CATALOG_PATH/index.json > /dev/null \
    && echo 'index.json valid after restore' \
    || { echo 'ERROR: restore produced invalid JSON'; exit 1; }

  # Count entries
  python3 -c \"
import json
with open('$CATALOG_PATH/index.json') as f: d = json.load(f)
print(f'Entries: {len(d.get(\\\"agents\\\", d.get(\\\"tools\\\", [])))}')
\"
"
```

### Verify recovery

```bash
# If Cortex serves this catalog, check it responds
curl -sf https://YOUR_CORTEX_HOST/catalog/index.json | python3 -m json.tool | head -5
# Should return valid JSON with non-zero entry count
```

### If .bak is also broken

Escalate to git history:

```bash
ssh rabi@179.61.138.8 "
  cd /srv/dr/kviz-public
  # Find the last good commit
  git log --oneline catalog/index.json | head -5
  # Restore from the last good commit
  git checkout HEAD~1 -- catalog/index.json
  python3 -m json.tool catalog/index.json > /dev/null && echo 'Git restore: valid'
"
```

---

## Rollback Surface 2 — Agent Files

### When to use
- Agent behaviour changed unexpectedly after an update
- Agent fails to load (tools.json error, wrong format)
- You need to revert a cortex.md model ID change
- `agent.md` line count dropped below 150 after edit

### Locate the backup

```bash
TARGET_HOST="rabi@179.61.138.8"
AGENT_ID="my-agent"

ssh $TARGET_HOST "
  ls -la ~/.config/opencode/agents/$AGENT_ID/
  # Look for: agent.md.bak, tools.json.bak, cortex.md.bak
"
```

### Restore agent.md

```bash
ssh $TARGET_HOST "
  DIR=~/.config/opencode/agents/$AGENT_ID
  [[ -f \$DIR/agent.md.bak ]] || { echo 'No .bak found'; exit 1; }

  # Check .bak line count
  BAK_LINES=\$(wc -l < \$DIR/agent.md.bak)
  echo \".bak has \$BAK_LINES lines\"
  [[ \$BAK_LINES -lt 150 ]] && echo 'WARNING: .bak is also under 150 lines'

  # Atomic restore
  cp \$DIR/agent.md.bak \$DIR/agent.md.tmp
  mv \$DIR/agent.md.tmp \$DIR/agent.md
  echo 'Restored agent.md'
  wc -l \$DIR/agent.md
"
```

### Restore tools.json

```bash
ssh $TARGET_HOST "
  DIR=~/.config/opencode/agents/$AGENT_ID
  [[ -f \$DIR/tools.json.bak ]] || { echo 'No .bak found'; exit 1; }

  # Validate backup JSON
  python3 -m json.tool \$DIR/tools.json.bak > /dev/null \
    && echo '.bak JSON valid' \
    || { echo 'ERROR: .bak invalid JSON'; exit 1; }

  cp \$DIR/tools.json.bak \$DIR/tools.json.tmp
  mv \$DIR/tools.json.tmp \$DIR/tools.json
  echo 'Restored tools.json'
  python3 -m json.tool \$DIR/tools.json | head -5
"
```

### Restore on both VPSes

```bash
AGENT_ID="my-agent"
for TARGET in rabi@179.61.138.8 rabi@179.61.251.98; do
  echo "=== Rolling back $AGENT_ID on $TARGET ==="
  ssh $TARGET "
    DIR=~/.config/opencode/agents/$AGENT_ID
    for f in agent.md tools.json cortex.md; do
      if [[ -f \$DIR/\$f.bak ]]; then
        cp \$DIR/\$f.bak \$DIR/\$f.tmp && mv \$DIR/\$f.tmp \$DIR/\$f
        echo \"Restored \$f\"
      else
        echo \"No .bak for \$f — skipping\"
      fi
    done
  "
done
```

---

## Rollback Surface 3 — Tool Directory

### When to use
- A new tool was added and broke the catalog or the vault loader
- A tool update introduced a bad `MANIFEST.json`
- Need to remove a tool entirely and restore the index

### Remove the bad tool and rebuild

```bash
TARGET_HOST="rabi@179.61.138.8"
TOOL_ID="bad-tool"
TOOLS_ROOT="/srv/dr/kviz-public/tools"

ssh $TARGET_HOST "
  set -e
  # Back up the tool dir before removing
  mv $TOOLS_ROOT/$TOOL_ID $TOOLS_ROOT/_removed-$TOOL_ID-\$(date +%Y%m%d) 2>/dev/null || true

  # Rebuild the index (tool will be absent now)
  python3 - <<'PYEOF'
import json, os
from datetime import datetime, timezone
tools = []
for entry in sorted(os.listdir('$TOOLS_ROOT')):
    if entry.startswith('_'): continue  # skip archived dirs
    mpath = os.path.join('$TOOLS_ROOT', entry, 'MANIFEST.json')
    if not os.path.isfile(mpath): continue
    m = json.load(open(mpath))
    tools.append({
        'id': m.get('id', entry),
        'name': m.get('name', entry),
        'version': m.get('version', '0.0.1'),
        'category': m.get('category', 'util'),
        'description': m.get('description', '')[:120],
        'path': f'tools/{entry}/',
        'entry': m.get('entry', 'main.sh'),
        'manifest': f'tools/{entry}/MANIFEST.json',
        'tags': m.get('tags', []),
        'requires': m.get('requires', []),
        'last_verified': m.get('updated', datetime.now(timezone.utc).date().isoformat())
    })
idx = {'version': '1.0', 'updated_at': datetime.now(timezone.utc).isoformat(), 'tools': tools}
with open('$TOOLS_ROOT/tools-index.json', 'w') as f:
    json.dump(idx, f, indent=2)
print(f'Rebuilt: {len(tools)} tools')
PYEOF

  # Verify
  python3 -m json.tool $TOOLS_ROOT/tools-index.json > /dev/null && echo 'tools-index.json valid'
  python3 -c \"import json; d=json.load(open('$TOOLS_ROOT/tools-index.json')); print(f'Tools: {len(d[\\\"tools\\\"])}')\"
"
```

### Restore a previously removed tool

```bash
ssh $TARGET_HOST "
  # Find the archived tool dir
  ls /srv/dr/kviz-public/tools/_removed-$TOOL_ID-*
  # Restore the most recent archive
  ARCHIVE=\$(ls -d /srv/dr/kviz-public/tools/_removed-$TOOL_ID-* | sort | tail -1)
  mv \$ARCHIVE /srv/dr/kviz-public/tools/$TOOL_ID
  echo 'Restored tool directory'
  # Then rebuild index
  python3 /srv/dr/scripts/rebuild-tools-index.py
"
```

---

## Rollback Surface 4 — Stamp / Version Marker

### When to use
- A version stamp file (`VERSION`, `STAMP`, `deployed_at`) was incorrectly updated
- A deploy script wrote a wrong version number that downstream checks read
- Need to revert a `CHANGELOG.md` entry

### Stamp files and their locations

| Stamp | Path | Format |
|---|---|---|
| Catalog version | `/srv/dr/kviz-public/catalog/VERSION` | `semver\nISO-date` |
| Tools vault stamp | `/srv/dr/kviz-public/tools/STAMP` | `ISO-datetime` |
| Brain last-updated | `/srv/dr/brains/_ops/LAST_UPDATED` | `ISO-datetime` |
| Deploy stamp | `/srv/dr/projects/<name>/deployed_at` | `ISO-datetime` |

### Restore a stamp

```bash
TARGET_HOST="rabi@179.61.138.8"
STAMP_PATH="/srv/dr/kviz-public/catalog/VERSION"

ssh $TARGET_HOST "
  [[ -f ${STAMP_PATH}.bak ]] || { echo 'No .bak for stamp'; exit 1; }
  cp ${STAMP_PATH}.bak ${STAMP_PATH}.tmp
  mv ${STAMP_PATH}.tmp $STAMP_PATH
  echo 'Restored stamp:'
  cat $STAMP_PATH
"
```

---

## Full Rollback Decision Tree

```
Something is broken after a change
            ↓
What surface?
  ├─ catalog / index.json   → Surface 1 (catalog rollback)
  ├─ agent not loading      → Surface 2 (agent rollback)
  ├─ tool broke the vault   → Surface 3 (tool directory rollback)
  └─ version stamp wrong    → Surface 4 (stamp rollback)
            ↓
Does a .bak exist on disk?
  ├─ YES → restore .bak, validate, confirm downstream recovery
  └─ NO  → git checkout HEAD~1 -- <file> ; validate ; confirm
            ↓
Is downstream service healthy?
  ├─ YES → rollback complete; document what went wrong
  └─ NO  → escalate; check if a second artifact also needs rollback
```

---

## Post-Rollback — What to Document

After every rollback, write a root-cause entry to the brain:

```
File: /srv/dr/brains/cybersec/knowledge/rollback-log.md (or web-dev equivalent)

## YYYY-MM-DD — <surface> rollback

**Symptom**: what broke, how it was detected  
**Root cause**: what change caused it  
**Rollback taken**: which .bak was restored / which git commit  
**Verification**: command run + output that confirmed recovery  
**Prevention**: what to change in the deploy process to avoid recurrence  
```

---

## Pre-Deploy Best Practices (prevent needing rollback)

Before any deploy or rebuild that overwrites a file:

```bash
# Always create .bak first
cp /path/to/file.json /path/to/file.json.bak
echo "Backup created: $(date -u)"

# Validate the NEW artifact before overwriting
python3 -m json.tool new-file.json > /dev/null || { echo "NEW file invalid — abort"; exit 1; }

# Then deploy
mv new-file.json /path/to/file.json
echo "Deployed. .bak preserved for rollback."
```

---

## Failure Modes and Fixes

| Symptom | Cause | Fix |
|---|---|---|
| `.bak` not found | Deploy skipped the backup step | Check git history: `git log --oneline <file>` |
| `.bak` is also invalid | Previous deploy was already broken | Roll back two commits: `git checkout HEAD~2 -- <file>` |
| `mv` not atomic (cross-FS) | Source and dest on different filesystems | Use `cp + rm` with explicit validation between steps |
| Rollback didn't fix behaviour | Two files both broken | Check all related files; catalog + stamp may both need rollback |
| Service still returning 500 after restore | Process cached old data | Restart the service / reload Cortex |

---

## Related Skills

- `vault-index-maintainer` — rebuilds tools-index.json; always creates .bak before rebuild
- `agent-lifecycle-manager` — always creates agent.md.bak before update
- `cve-tracker` — does not modify deployed files; no rollback surface
- `payload-curator` — git-managed; rollback via `git checkout`
