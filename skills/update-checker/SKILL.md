# update-checker

## Purpose
Hash-diff check to detect whether the local skill catalog, vault files, or fleet prefix are stale compared to the authoritative remote copies. Runs at session startup or on demand. Tells you exactly what changed, what is current, and whether to invoke pull-sync.

## Trigger Conditions
- Session startup on any machine (PC, web-dev VPS 179.61.138.8, knowledge VPS 179.61.251.98)
- User says: "check for updates", "is the catalog stale", "what changed", "sync check", "update check"
- Before any cross-fleet orchestration run (prefix-sync step)
- After a known deployment to confirm the update landed

---

## What to Check

| Artifact | Local stamp location | Remote authority | Method |
|---|---|---|---|
| Skill catalog | `~/.config/opencode/skills/__catalog-stamp.sha` | `deployrocket-central-brain` repo `skills/index.json` | SHA compare |
| Vault / brain | `~/.config/opencode/memory/__vault-stamp.sha` | `deployrocket-central-brain` repo HEAD | git `ls-remote HEAD` |
| Fleet prefix | `~/.config/opencode/skills/prefix/__prefix-stamp.sha` | Same repo `prefix/SKILL.md` blob SHA | SHA compare |

---

## Step-by-Step Procedure

### Step 1 — Read local stamps

```bash
# On Linux VPS (bash)
LOCAL_CATALOG=$(cat ~/.config/opencode/skills/__catalog-stamp.sha 2>/dev/null || echo "MISSING")
LOCAL_VAULT=$(cat ~/.config/opencode/memory/__vault-stamp.sha 2>/dev/null || echo "MISSING")
LOCAL_PREFIX=$(cat ~/.config/opencode/skills/prefix/__prefix-stamp.sha 2>/dev/null || echo "MISSING")
```

```powershell
# On Windows PC / VPS (PowerShell)
$LocalCatalog = Get-Content "$env:USERPROFILE\.config\opencode\skills\__catalog-stamp.sha" -ErrorAction SilentlyContinue
if (-not $LocalCatalog) { $LocalCatalog = "MISSING" }
$LocalVault   = Get-Content "$env:USERPROFILE\.config\opencode\memory\__vault-stamp.sha"  -ErrorAction SilentlyContinue
if (-not $LocalVault)   { $LocalVault   = "MISSING" }
$LocalPrefix  = Get-Content "$env:USERPROFILE\.config\opencode\skills\prefix\__prefix-stamp.sha" -ErrorAction SilentlyContinue
if (-not $LocalPrefix)  { $LocalPrefix  = "MISSING" }
```

### Step 2 — Fetch remote SHAs via GitHub API (no clone needed)

```bash
REPO="RabiskaPiska/deployrocket-central-brain"
REMOTE_HEAD=$(curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/$REPO/git/refs/heads/main" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['object']['sha'][:12])")

REMOTE_CATALOG=$(curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/$REPO/contents/skills/index.json" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['sha'][:12])")

REMOTE_PREFIX=$(curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/$REPO/contents/prefix/SKILL.md" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['sha'][:12])")
```

If `GITHUB_TOKEN` is unavailable, fall back to `tools.github.get_latest_release` or use `execute` with `tools.github` MCP — see mcp-health-checker for fallback chain.

### Step 3 — Compare and classify

For each artifact:
- `LOCAL == REMOTE` → **CURRENT** — no action
- `LOCAL == "MISSING"` → **NEVER SYNCED** — run full pull-sync
- `LOCAL != REMOTE` → **STALE** — run pull-sync

### Step 4 — Emit status table

```
UPDATE CHECK — 2026-10-31T14:22:00Z  (machine: web-dev-vps)
  skill catalog  : ✅ CURRENT  (local a3f9b1 == remote a3f9b1)
  vault / brain  : ⚠️  STALE   (local d4c2a0  != remote 9e1f33) — pull-sync needed
  fleet prefix   : ✅ CURRENT  (local 7bc401 == remote 7bc401)

ACTION: run pull-sync for vault only.
```

---

## What "Changed vs Unchanged" Looks Like

**Unchanged (CURRENT):**
- Local stamp SHA matches remote object SHA byte-for-byte (first 12 hex chars sufficient for daily checks)
- No pull-sync needed; proceed with the session

**Stale:**
- SHAs differ — something was committed to the central brain since this machine last synced
- Could be: new skills added, memory notes updated, prefix patched, CVE entries refreshed
- Run pull-sync; then re-stamp the local file with the new SHA

**Never synced (MISSING):**
- Stamp file does not exist — machine is new or was wiped
- Run full pull-sync to hydrate all artifacts, then create all stamp files

---

## Triggering pull-sync

After a STALE or MISSING result, invoke the pull-sync skill:

```
Skill("pull-sync")
```

Or manually on VPS:
```bash
cd /srv/dr && git pull origin main && bash brain-sync.sh
```

After sync completes, write the new SHA back to the stamp file:
```bash
echo "$REMOTE_HEAD" > ~/.config/opencode/memory/__vault-stamp.sha
echo "$REMOTE_CATALOG" > ~/.config/opencode/skills/__catalog-stamp.sha
echo "$REMOTE_PREFIX" > ~/.config/opencode/skills/prefix/__prefix-stamp.sha
```

---

## Session-Startup Fast Path

If all three stamps are CURRENT: emit one line — `Catalog/vault/prefix: all current` — and proceed. Do not slow the session with a verbose table when nothing needs action.

If any is STALE or MISSING: emit the full table, invoke pull-sync automatically (bypass-permissions is ON), re-stamp, then continue.

---

## Freshness SLA (fleet policy)

| Artifact | Expected sync cadence | Alert threshold |
|---|---|---|
| Skill catalog | Every deploy / weekly at minimum | >7 days stale |
| Vault / brain | Every substantive session | >48 hours stale |
| Fleet prefix | On every prefix update | Any mismatch |

If the vault stamp is older than 48 hours (compare stamp mtime to `date`), emit a warning even if the SHA matches — the stamp file may not have been updated after a manual edit.

---

## Notes
- Never trust the stamp file alone if you suspect a partial sync — re-verify the SHA.
- On PC, stamp paths use `%USERPROFILE%` not `~` — use PowerShell paths.
- GitHub API returns full 40-char SHA; truncate to 12 for display, compare full 40 for correctness.
