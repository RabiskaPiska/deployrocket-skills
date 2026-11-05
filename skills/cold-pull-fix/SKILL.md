---
name: cold-pull-fix
description: Fixes the pull-sync cold-path bug where index.json returns the path "agents/index.json" and that path resolves to a 404, causing the puller to exit with code 2 and leaving the local skill catalog stale. Triggers on pull-sync failures, "agents/index.json not found" errors, and any exit code 2 from pull-sync. Provides the patched logic, immediate workaround, and the verification test.
---

# Cold-Pull Fix

## Bug Summary

**Symptom:** `pull-sync` exits with code 2 and prints one of:

```
ERROR: agents/index.json not found (HTTP 404)
FileNotFoundError: agents/index.json
Pull failed: remote index unreachable (exit 2)
```

**Cause:** The pull-sync cold path — the branch that runs when no local cache exists yet —
fetches the top-level `index.json` from the remote. That file contains a `"path"` field that
points to the skills sub-index. On some remote configurations the value is the relative path
`agents/index.json` rather than the full URL or an absolute path. The puller naively joins this
relative string onto the base URL, producing a double-path like:

```
https://host/skills/agents/index.json   ← 404
```

instead of the correct:

```
https://host/agents/index.json          ← 200
```

The puller does not retry and does not fall back; it raises the error and exits 2.
Because this only fires on a cold pull (no local cache), it silently leaves new installs
with zero skills loaded and gives no actionable guidance to the operator.

---

## Trigger Conditions

Load this skill when:

- `pull-sync` exits with code 2
- Log contains `agents/index.json not found` or `HTTP 404` during a pull
- A fresh machine/VPS has no skills after running the standard install
- Skill count reported by `opencode skill list` is 0 after a sync

---

## Root Cause in Detail

### The bad cold path (original logic)

```python
# pull-sync.py  — original cold path (BROKEN)

def fetch_index(base_url: str) -> dict:
    resp = requests.get(f"{base_url}/index.json", timeout=10)
    resp.raise_for_status()
    meta = resp.json()
    # BUG: meta["path"] may be "agents/index.json" (relative)
    # joining it onto base_url produces a doubled path segment
    sub_index_url = f"{base_url}/{meta['path']}"   # <-- WRONG when path is relative
    sub = requests.get(sub_index_url, timeout=10)
    sub.raise_for_status()    # raises 404 here → exit 2
    return sub.json()
```

### Why it only fires on cold pulls

On warm pulls the local cache already contains the resolved skill list. The sub-index fetch
is skipped entirely. Only a cold pull (empty cache, first run, or `--force-fresh`) hits this
code path.

---

## The Fix

### Patched logic

```python
# pull-sync.py  — patched cold path

import urllib.parse, requests

def _resolve_url(base: str, path_or_url: str) -> str:
    """
    Resolve path_or_url relative to base.
    If path_or_url is already an absolute URL, return it unchanged.
    If it is a relative path, strip any leading slash and join onto the
    ORIGIN of base (scheme + host), NOT onto base itself, to avoid
    double-path bugs.
    """
    if path_or_url.startswith(("http://", "https://")):
        return path_or_url                          # already absolute
    parsed = urllib.parse.urlparse(base)
    origin = f"{parsed.scheme}://{parsed.netloc}"  # strip any existing path
    clean  = path_or_url.lstrip("/")
    return f"{origin}/{clean}"

def fetch_index(base_url: str) -> dict:
    resp = requests.get(f"{base_url}/index.json", timeout=10)
    resp.raise_for_status()
    meta = resp.json()

    sub_path = meta.get("path") or meta.get("skills_path") or "agents/index.json"
    sub_index_url = _resolve_url(base_url, sub_path)   # safe resolution

    sub = requests.get(sub_index_url, timeout=10)
    if sub.status_code == 404:
        # Graceful degradation: log and return empty catalog rather than crashing
        print(f"WARN: sub-index not found at {sub_index_url}; returning empty catalog",
              file=sys.stderr)
        return {"skills": []}
    sub.raise_for_status()
    return sub.json()
```

### Key changes

| Change | Reason |
|--------|--------|
| `_resolve_url` strips existing path from base before joining | Prevents double-path segment |
| Absolute URL passthrough | Handles remotes that already return a full URL in `"path"` |
| `meta.get("skills_path")` fallback key | Some remotes use a different field name |
| 404 graceful degradation | Returns empty catalog instead of exit 2; operator sees a warning, not a crash |

---

## Immediate Workaround (before patching)

If you need skills loaded right now and cannot patch pull-sync yet:

### Option A — Manual sub-index fetch

```bash
BASE="https://your-remote-host"
# Fetch top-level index to find the real sub-index path
curl -s "$BASE/index.json" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('path','agents/index.json'))"
# Example output: agents/index.json

# Fetch the sub-index directly from the ORIGIN (not from $BASE/...)
curl -s "$BASE/agents/index.json" -o /tmp/skill-catalog.json
# If that 404s, try:
curl -s "$BASE/index.json" -o /tmp/skill-catalog.json
```

### Option B — Force local cache seed

If you have a known-good catalog from another machine:

```bash
# On the working machine
scp ~/.config/opencode/skills/catalog.json user@new-host:~/.config/opencode/skills/

# On the new host — run warm pull (skips cold path entirely)
opencode pull-sync --warm
```

### Option C — Skip pull-sync, install manually

```bash
# Clone skills directly and register
git clone https://your-remote-host/skills.git ~/.config/opencode/skills/
opencode skill scan ~/.config/opencode/skills/
```

---

## Verification Test

After applying the patch, confirm the cold path works:

```bash
#!/usr/bin/env bash
# test-cold-pull.sh — run this to verify the fix

set -e

echo "=== Cold-pull fix verification ==="

# 1. Wipe local cache to force a cold pull
CACHE_DIR="${OPENCODE_CONFIG_DIR:-$HOME/.config/opencode}/skills"
BACKUP_DIR="/tmp/skills-backup-$(date +%s)"
if [ -d "$CACHE_DIR" ]; then
    mv "$CACHE_DIR" "$BACKUP_DIR"
    echo "Cache backed up to $BACKUP_DIR"
fi

# 2. Run pull-sync and capture exit code
set +e
opencode pull-sync 2>&1 | tee /tmp/pull-sync-test.log
EXIT_CODE=$?
set -e

# 3. Evaluate
if [ $EXIT_CODE -eq 0 ]; then
    COUNT=$(opencode skill list 2>/dev/null | wc -l)
    echo "PASS: pull-sync exited 0; $COUNT skills loaded"
elif grep -q "agents/index.json not found" /tmp/pull-sync-test.log; then
    echo "FAIL: cold-path bug still present (exit $EXIT_CODE)"
    echo "Apply the patch to pull-sync.py and retry."
    # Restore backup so the machine is not left skill-less
    [ -d "$BACKUP_DIR" ] && mv "$BACKUP_DIR" "$CACHE_DIR"
    exit 1
else
    echo "FAIL: pull-sync exited $EXIT_CODE for an unrelated reason"
    cat /tmp/pull-sync-test.log
    [ -d "$BACKUP_DIR" ] && mv "$BACKUP_DIR" "$CACHE_DIR"
    exit 1
fi

# 4. Restore backup if needed (optional — remove if you want the fresh pull to persist)
# mv "$BACKUP_DIR" "$CACHE_DIR"

echo "=== Verification complete ==="
```

Expected output after fix:
```
=== Cold-pull fix verification ===
Cache backed up to /tmp/skills-backup-XXXXXXXXXX
PASS: pull-sync exited 0; 92 skills loaded
=== Verification complete ===
```

---

## Fleet Deployment

Apply the patch on all machines that run pull-sync:

```bash
# On each VPS / PC
PULL_SYNC=$(find / -name "pull-sync.py" 2>/dev/null | head -1)
echo "Found at: $PULL_SYNC"

# Backup original
cp "$PULL_SYNC" "${PULL_SYNC}.bak-$(date +%Y%m%d)"

# Apply patch (substitute with actual patched file path)
cp /path/to/patched-pull-sync.py "$PULL_SYNC"

# Verify
bash test-cold-pull.sh
```

---

## Prevention

1. **Pin the `"path"` field format in the remote index schema** — require it to be an absolute
   URL or a root-relative path starting with `/`. Document this in the remote's README.

2. **Add the 404 graceful degradation** regardless of whether the double-path bug is present —
   a missing sub-index should never hard-crash the puller; an empty catalog with a warning is
   always preferable to exit 2.

3. **Add `test-cold-pull.sh` to CI** on any machine that provisions new OpenCode installs,
   so a schema change on the remote does not silently break new deployments.

---

## Related Issues

- Pull-sync warm path: unaffected (bypasses `fetch_index`)
- Pull-sync auth failures: different exit code (3), different error message
- Skill scan failures after a successful pull: separate issue — check file permissions on
  `~/.config/opencode/skills/`
