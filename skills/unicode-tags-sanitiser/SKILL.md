---
name: unicode-tags-sanitiser
description: Prevents OpenCode runtime crashes caused by Unicode Tags block characters (U+E0000–U+E007F) silently embedded in SKILL.md files. Triggers on every SKILL.md write, any skill authoring task, and any pull-sync that imports external skill files. Provides detection, sanitisation, safe substitution rules, and a CI gate command.
---

# Unicode Tags Sanitiser

## Why This Exists

OpenCode parses SKILL.md files at skill-load time. Characters in the Unicode Tags block
(U+E0000–U+E007F) are invisible in most editors and terminals but cause the YAML front-matter
parser and the downstream skill renderer to throw an unhandled exception, crashing the runtime
with no useful error message. The crash is silent from the author's perspective because the
characters do not appear in `cat`, `less`, or most file-diff views.

Common sources:
- LLM completions that encode control metadata as invisible tag characters
- Copy-paste from certain PDF viewers or browser clipboard APIs
- Base64-decoded payloads embedded in skill content by a compromised upstream

---

## Trigger Conditions

Fire this skill automatically when:
- Writing any new SKILL.md file
- Editing an existing SKILL.md file
- Running pull-sync that imports skill files from a remote source
- A skill load fails with a YAML or encoding error and the cause is unclear

---

## Detection

### Quick shell check (PowerShell)

```powershell
# Detect Unicode Tags block characters in a file
$bytes = [System.IO.File]::ReadAllBytes("path\to\SKILL.md")
$text  = [System.Text.Encoding]::UTF8.GetString($bytes)
$hits  = ($text.ToCharArray() | Where-Object { [int]$_ -ge 0xE0000 -and [int]$_ -le 0xE007F }).Count
if ($hits -gt 0) { Write-Error "BLOCKED: $hits Unicode Tags character(s) found"; exit 1 }
else             { Write-Host "CLEAN: no Unicode Tags characters" }
```

### Quick shell check (bash / Linux VPS)

```bash
# Requires Python 3; exits 1 if tags block characters are present
python3 - "$1" <<'EOF'
import sys, pathlib
text = pathlib.Path(sys.argv[1]).read_text(encoding="utf-8", errors="replace")
hits = [c for c in text if 0xE0000 <= ord(c) <= 0xE007F]
if hits:
    print(f"BLOCKED: {len(hits)} Unicode Tags character(s) at codepoints: "
          f"{[hex(ord(c)) for c in hits[:10]]}", file=sys.stderr)
    sys.exit(1)
print("CLEAN")
EOF
```

### Scan entire skills directory

```bash
# Prints each dirty file and its hit count; exits non-zero if any found
find . -name "SKILL.md" | while read f; do
  python3 -c "
import sys, pathlib
t = pathlib.Path('$f').read_text(encoding='utf-8', errors='replace')
h = [c for c in t if 0xE0000 <= ord(c) <= 0xE007F]
if h: print(f'DIRTY {len(h):4d}  $f')
"
done
```

---

## Sanitisation Procedure

### Step 1 — Strip invisibles (Python, safe in-place)

```python
import pathlib, re, sys

def strip_unicode_tags(path: str) -> int:
    """Remove all Unicode Tags block chars. Returns count removed."""
    p = pathlib.Path(path)
    original = p.read_text(encoding="utf-8", errors="replace")
    cleaned  = re.sub(r"[\U000E0000-\U000E007F]", "", original)
    removed  = len(original) - len(cleaned)
    if removed:
        p.write_text(cleaned, encoding="utf-8")
        print(f"Removed {removed} character(s) from {path}")
    return removed

if __name__ == "__main__":
    for f in sys.argv[1:]:
        strip_unicode_tags(f)
```

### Step 2 — Verify the fix

Re-run the detection check. It must print `CLEAN` before the file is committed or loaded.

### Step 3 — Commit only clean files

Add the detection script as a pre-commit hook or CI gate (see below).

---

## Safe Substitutions

If the Unicode Tags characters were intentionally encoding something (rare but possible in
auto-generated skills), use these safe ASCII/Unicode substitutions:

| Intent                  | Unsafe encoding       | Safe replacement          |
|-------------------------|-----------------------|---------------------------|
| Language tag (e.g. `en`)| U+E0065 U+E006E       | Plain ASCII `en`          |
| Cancel tag              | U+E007F               | Remove entirely           |
| Any invisible metadata  | Tags block chars      | YAML front-matter key     |

Rule: **never** use Tags block characters for semantic content. Store metadata in YAML
front-matter or explicit Markdown sections only.

---

## CI Gate

### GitHub Actions step

```yaml
- name: Check SKILL.md files for Unicode Tags block
  run: |
    python3 - <<'EOF'
    import sys, pathlib, glob
    dirty = []
    for f in glob.glob("**/.opencode/skills/**/SKILL.md", recursive=True):
        t = pathlib.Path(f).read_text(encoding="utf-8", errors="replace")
        h = [c for c in t if 0xE0000 <= ord(c) <= 0xE007F]
        if h:
            dirty.append((f, len(h)))
    if dirty:
        for f, n in dirty:
            print(f"::error file={f}::Unicode Tags block: {n} character(s)")
        sys.exit(1)
    print(f"All SKILL.md files clean.")
    EOF
```

### Pre-commit hook (`.git/hooks/pre-commit`)

```bash
#!/usr/bin/env bash
set -e
python3 - $(git diff --cached --name-only | grep 'SKILL\.md') <<'EOF'
import sys, pathlib
for f in sys.argv[1:]:
    t = pathlib.Path(f).read_text(encoding="utf-8", errors="replace")
    h = [c for c in t if 0xE0000 <= ord(c) <= 0xE007F]
    if h:
        print(f"BLOCKED: {f} contains {len(h)} Unicode Tags char(s)", file=sys.stderr)
        sys.exit(1)
EOF
```

---

## Authoring Checklist

Before saving any SKILL.md:

- [ ] Run the detection script; confirm output is `CLEAN`
- [ ] Open the file in a hex editor or `xxd | grep 'e0 00'` and confirm no E0xxx bytes
- [ ] If the file was generated by an LLM, always run detection — LLMs can silently embed Tags chars
- [ ] Commit only after the CI gate passes

---

## Recovery — Runtime Already Crashed

1. Identify the offending skill: the crash usually names a file in the stack trace. If not,
   bisect by temporarily removing half the skills and reloading.
2. Run the sanitisation script on the identified file.
3. Reload the skill: `opencode skill reload <id>` or restart the session.
4. Add the CI gate to prevent recurrence.
