# vault-index-maintainer

**Manages the tool vault index — tools-index.json schema, scanning, rebuilding, MANIFEST.json format.**

## Triggers

- After any tool is added, updated, or removed from `/srv/dr/kviz-public/tools/`
- "update vault index"
- "rebuild tools-index.json"
- "add this tool to the catalog"
- "sync the vault index"
- "tool not showing in catalog"

---

## Context

The tool vault lives at `/srv/dr/kviz-public/tools/` on `179.61.138.8`. Every tool is a subdirectory. The top-level `tools-index.json` is the discovery file that Cortex and downstream agents query. It must always reflect the current on-disk state — stale or missing entries cause silent tool-call failures.

---

## tools-index.json Schema

```json
{
  "version": "1.0",
  "updated_at": "ISO-8601 timestamp",
  "tools": [
    {
      "id": "tool-slug",
      "name": "Human-readable name",
      "version": "semver e.g. 1.0.0",
      "category": "recon|exploit|post-exploit|util|web|crypto|network|osint",
      "description": "One sentence. What it does and when to use it.",
      "path": "tools/<tool-slug>/",
      "entry": "main.sh or main.py or main.go",
      "manifest": "tools/<tool-slug>/MANIFEST.json",
      "tags": ["array", "of", "keywords"],
      "requires": ["dependency1", "dependency2"],
      "last_verified": "ISO-8601 date"
    }
  ]
}
```

Field rules:
- `id` — lowercase, hyphen-separated, no spaces, matches directory name exactly
- `version` — bump the patch digit on any content change; minor on new features; major on breaking interface change
- `category` — must be one of the allowed values above; use `util` for generic helpers
- `description` — under 120 characters; no marketing language
- `entry` — the file that gets executed when the tool runs; must exist on disk
- `requires` — list apt/pip/npm packages; empty array `[]` if self-contained
- `last_verified` — date the tool was last test-run and confirmed working; update on every successful run

---

## MANIFEST.json Format (per-tool)

Every tool directory **must** contain a `MANIFEST.json`. Vault index rebuild reads this file; a missing manifest causes the tool to be skipped with a warning.

```json
{
  "id": "tool-slug",
  "name": "Human-readable name",
  "version": "1.0.0",
  "author": "seat-id or handle",
  "created": "ISO-8601 date",
  "updated": "ISO-8601 date",
  "category": "recon",
  "description": "Full description — up to 500 characters.",
  "entry": "main.sh",
  "args": [
    {
      "name": "target",
      "type": "string",
      "required": true,
      "description": "Target hostname or IP"
    }
  ],
  "outputs": ["stdout", "report.txt"],
  "requires": [],
  "tested_on": ["ubuntu-24.04"],
  "license": "MIT",
  "tags": []
}
```

---

## Scanning and Rebuilding tools-index.json

### Full rebuild (safe — always do this, never hand-edit the index)

```bash
#!/usr/bin/env bash
# Run on web-dev VPS as rabi
TOOLS_ROOT="/srv/dr/kviz-public/tools"
INDEX_FILE="$TOOLS_ROOT/tools-index.json"
BACKUP="$INDEX_FILE.bak"

# 1. Back up current index before touching it
[[ -f "$INDEX_FILE" ]] && cp "$INDEX_FILE" "$BACKUP"

# 2. Collect all tool dirs that have a MANIFEST.json
python3 - <<'PYEOF'
import json, os, sys
from datetime import datetime, timezone

tools_root = "/srv/dr/kviz-public/tools"
tools = []

for entry in sorted(os.listdir(tools_root)):
    tool_dir = os.path.join(tools_root, entry)
    manifest_path = os.path.join(tool_dir, "MANIFEST.json")
    if not os.path.isdir(tool_dir) or not os.path.isfile(manifest_path):
        print(f"WARN: skipping {entry} — no MANIFEST.json", file=sys.stderr)
        continue
    try:
        with open(manifest_path) as f:
            m = json.load(f)
    except json.JSONDecodeError as e:
        print(f"ERROR: {entry}/MANIFEST.json invalid JSON — {e}", file=sys.stderr)
        continue
    # Build index entry from manifest
    tools.append({
        "id": m.get("id", entry),
        "name": m.get("name", entry),
        "version": m.get("version", "0.0.1"),
        "category": m.get("category", "util"),
        "description": m.get("description", "")[:120],
        "path": f"tools/{entry}/",
        "entry": m.get("entry", "main.sh"),
        "manifest": f"tools/{entry}/MANIFEST.json",
        "tags": m.get("tags", []),
        "requires": m.get("requires", []),
        "last_verified": m.get("updated", datetime.now(timezone.utc).date().isoformat())
    })

index = {
    "version": "1.0",
    "updated_at": datetime.now(timezone.utc).isoformat(),
    "tools": tools
}
out_path = os.path.join(tools_root, "tools-index.json")
with open(out_path, "w") as f:
    json.dump(index, f, indent=2)
print(f"OK: wrote {len(tools)} tools to {out_path}")
PYEOF
```

### Validate after rebuild

```bash
python3 -c "
import json
with open('/srv/dr/kviz-public/tools/tools-index.json') as f:
    idx = json.load(f)
print(f'tools-index.json: {len(idx[\"tools\"])} tools, updated {idx[\"updated_at\"]}')
for t in idx['tools']:
    entry = f'/srv/dr/kviz-public/{t[\"path\"]}{t[\"entry\"]}'
    exists = '✓' if __import__(\"os\").path.isfile(entry) else '✗ MISSING'
    print(f'  {exists}  {t[\"id\"]}  v{t[\"version\"]}')
"
```

---

## Adding a New Tool — Checklist

1. Create the directory: `mkdir -p /srv/dr/kviz-public/tools/<tool-slug>/`
2. Write the tool entry point (`main.sh`, `main.py`, etc.)
3. Write `MANIFEST.json` following the schema above
4. Run the rebuild script (section above)
5. Run the validate script — confirm `✓` appears next to the new tool
6. Commit: `cd /srv/dr/kviz-public && git add -A && git commit -m "tools: add <tool-slug>"`

## Removing a Tool — Checklist

1. `rm -rf /srv/dr/kviz-public/tools/<tool-slug>/`
2. Run the rebuild script — the tool will no longer appear (no MANIFEST = skipped)
3. Run the validate script — confirm tool is gone
4. Commit the removal

## Updating a Tool — Checklist

1. Edit the files inside the tool directory
2. Bump `version` in `MANIFEST.json` and update `updated` date
3. Run rebuild → validate
4. Commit

---

## Failure modes and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Tool not in catalog after add | MANIFEST.json missing or invalid JSON | Validate JSON: `python3 -m json.tool MANIFEST.json` |
| `entry` file not found in validate | Wrong filename in `entry` field | Match `entry` in MANIFEST.json to the real filename |
| index.json not updated after rebuild | Rebuild script had a Python exception | Check stderr output from the rebuild; fix the bad MANIFEST |
| Old version still showing | index.json not committed | `git add -A && git commit` |
| Catalog query returns stale data | Cortex cached old index | Wait TTL or force-reload via Cortex API |

---

## Recovery — restore from backup

```bash
cp /srv/dr/kviz-public/tools/tools-index.json.bak \
   /srv/dr/kviz-public/tools/tools-index.json
echo "restored from backup"
python3 -m json.tool /srv/dr/kviz-public/tools/tools-index.json > /dev/null && echo "JSON valid"
```

---

## Related skills

- `rollback-executor` — restores index from .bak when a bad rebuild ships
- `agent-lifecycle-manager` — parallel pattern for agent versioning
