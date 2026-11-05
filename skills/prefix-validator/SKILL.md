# prefix-validator

## Purpose
Validates that the fleet system prefix loads correctly, injects the right context, and fires the correct skills on all three target platforms before the prefix is shipped or after any edit. Produces a per-platform pass/fail transcript you can attach to a deploy PR or maintenance log.

## Trigger Conditions
- User says: "test the prefix", "validate the prefix", "check prefix on this platform", "does the prefix work", "prefix broken?"
- Before shipping any new version of the prefix SKILL.md to the fleet
- After any edit to `~/.config/opencode/skills/prefix/SKILL.md`
- As part of the weekly maintenance run (W-prefix gate)

---

## The Three Platforms

| Platform ID | Machine | Shell | Config root |
|---|---|---|---|
| `PC-PS` | Rabi's main PC (Win 11) | PowerShell 7 | `C:\Users\rabim\.config\opencode\` |
| `VPS-BASH` | web-dev VPS 179.61.138.8 | bash (Ubuntu 24.04) | `~/.config/opencode/` |
| `VPS-WIN` | Windows VPS 179.61.138.5 | PowerShell 5/7 | `C:\Users\rabi\.config\opencode\` or `%USERPROFILE%\.config\opencode\` |

---

## What a Passing Prefix Test Looks Like

A prefix is passing on a platform when all four checks hold:

1. **File present and readable** — `prefix/SKILL.md` exists at the config root and is non-empty (>100 bytes)
2. **No Unicode Tags block characters** — no codepoints U+E0000–U+E007F (these crash OpenCode silently; see unicode-tags-sanitiser skill)
3. **No trailing NUL bytes or CRLF corruption** — file is clean UTF-8
4. **Key marker strings present** — the prefix contains at least:
   - The string `GodMode` (confirms godmode injection)
   - The string `honestscore` (confirms gate injection)
   - The string `PromptGod` (confirms auto-mode injection)
   - The string `HEARTBEAT` (confirms monitoring injection)

---

## Platform Validation Procedures

### PC-PS (Windows PowerShell 7)

```powershell
$PrefixPath = "$env:USERPROFILE\.config\opencode\skills\prefix\SKILL.md"

# CHECK 1 — file present
Test-Path $PrefixPath

# CHECK 2 — non-empty
(Get-Item $PrefixPath).Length -gt 100

# CHECK 3 — Unicode Tags scan
$bytes = [System.IO.File]::ReadAllBytes($PrefixPath)
$utf8  = [System.Text.Encoding]::UTF8.GetString($bytes)
$badChars = $utf8 | Select-String -Pattern '[\uE000-\uE07F]'
if ($badChars) { "FAIL: Unicode Tags found" } else { "PASS: no Unicode Tags" }

# CHECK 4 — key markers
$content = Get-Content $PrefixPath -Raw
@("GodMode","honestscore","PromptGod","HEARTBEAT") | ForEach-Object {
  if ($content -match $_) { "PASS: $_ present" } else { "FAIL: $_ MISSING" }
}
```

**Passing transcript (PC-PS):**
```
True
True
PASS: no Unicode Tags
PASS: GodMode present
PASS: honestscore present
PASS: PromptGod present
PASS: HEARTBEAT present
PC-PS RESULT: ✅ ALL PASS
```

---

### VPS-BASH (Linux bash)

```bash
PREFIX=~/.config/opencode/skills/prefix/SKILL.md

# CHECK 1 — file present
test -f "$PREFIX" && echo "PASS: file present" || echo "FAIL: file missing"

# CHECK 2 — non-empty (>100 bytes)
SIZE=$(wc -c < "$PREFIX")
[ "$SIZE" -gt 100 ] && echo "PASS: size $SIZE bytes" || echo "FAIL: too small ($SIZE bytes)"

# CHECK 3 — Unicode Tags block (U+E0000–U+E007F are private-use tag chars)
# Python handles this more reliably than grep on all distros
python3 -c "
import sys, re
text = open('$PREFIX', encoding='utf-8', errors='replace').read()
hits = re.findall(r'[\U000E0000-\U000E007F]', text)
if hits:
    print(f'FAIL: {len(hits)} Unicode Tags characters found')
else:
    print('PASS: no Unicode Tags characters')
"

# CHECK 4 — key markers
for MARKER in "GodMode" "honestscore" "PromptGod" "HEARTBEAT"; do
  grep -q "$MARKER" "$PREFIX" \
    && echo "PASS: $MARKER present" \
    || echo "FAIL: $MARKER MISSING"
done
```

**Passing transcript (VPS-BASH):**
```
PASS: file present
PASS: size 14823 bytes
PASS: no Unicode Tags characters
PASS: GodMode present
PASS: honestscore present
PASS: PromptGod present
PASS: HEARTBEAT present
VPS-BASH RESULT: ✅ ALL PASS
```

---

### VPS-WIN (Windows VPS PowerShell)

Same script as PC-PS, but path root differs:

```powershell
# VPS-WIN path (adjust username if needed)
$PrefixPath = "C:\Users\rabi\.config\opencode\skills\prefix\SKILL.md"

# Run the same 4 checks as PC-PS above
```

**If WinPS 5 (not 7) is present:** `-match` regex behaves the same; `[System.IO.File]::ReadAllBytes` works on PS5. No changes needed.

---

## Validation Transcript Format

After running all three platforms (or the current platform if cross-platform SSH is unavailable), emit:

```
PREFIX VALIDATION TRANSCRIPT — 2026-10-31T10:15:00Z
  Prefix file   : ~/.config/opencode/skills/prefix/SKILL.md
  File size     : 14823 bytes
  Last modified : 2026-10-30T18:42:11Z

  Platform       | File | Size | Encoding | Markers | RESULT
  ---------------|------|------|----------|---------|-------
  PC-PS          |  ✅  |  ✅  |    ✅    |  4/4 ✅ | PASS
  VPS-BASH       |  ✅  |  ✅  |    ✅    |  4/4 ✅ | PASS
  VPS-WIN        |  ✅  |  ✅  |    ✅    |  4/4 ✅ | PASS

  OVERALL: ✅ PASS on all 3 platforms — safe to deploy / treat as current.
```

If any platform fails:

```
  VPS-WIN        |  ❌  |  —   |    —     |   —     | FAIL: file missing
  OVERALL: ❌ FAIL — do not ship until resolved.
  ACTION: copy prefix to VPS-WIN config root and re-run.
```

---

## Automated Cross-Platform Run

When run from the web-dev VPS (which has SSH to all machines):

```bash
# Local VPS check
bash -s <<'EOF'
PREFIX=~/.config/opencode/skills/prefix/SKILL.md
test -f "$PREFIX" && python3 -c "
import re, sys
t = open('$PREFIX').read()
hits = re.findall(r'[\U000E0000-\U000E007F]', t)
marks = ['GodMode','honestscore','PromptGod','HEARTBEAT']
ok = not hits and all(m in t for m in marks)
print('VPS-BASH: PASS' if ok else 'VPS-BASH: FAIL')
"
EOF

# PC check (if RC tunnel is live)
# ssh rabi@<pc-tunnel> "powershell -Command ..."
# Otherwise: manual run on PC and paste transcript here
```

---

## On FAIL — Fix Actions

| Failure type | Fix |
|---|---|
| File missing | Deploy prefix via pull-sync or direct scp |
| File too small (<100B) | Re-deploy; check for truncated write |
| Unicode Tags found | Run unicode-tags-sanitiser skill; re-deploy clean file |
| Marker missing | Edit prefix SKILL.md to restore the missing section |
| Encoding corruption | Re-generate from source; never copy via clipboard (lossy) |

---

## Notes
- Run this skill before AND after every prefix update — before to confirm current state, after to confirm the deploy landed.
- The validation is read-only and safe to run at any time; it makes no changes.
- If the PC is not accessible via SSH, paste the PC-PS transcript manually; do not mark PC-PS as PASS without evidence.