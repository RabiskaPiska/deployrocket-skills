# payload-curator

**Manages payload libraries for pentest work — PayloadsAllTheThings vault structure, category lookup, WAF-bypass selection, encoding variants.**

## Triggers

- "give me payloads for X"
- "which payloads bypass Y WAF"
- "find XSS payloads that bypass Cloudflare"
- "SQLi payloads for MSSQL"
- "encoded variants of this payload"
- "update the payload library"
- "what's in the payload vault"
- Any pentest skill requesting raw payload lists

---

## Context

The payload vault lives on the web-dev VPS at `/srv/dr/payloads/` and mirrors the [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) structure. It is supplemented with curated WAF-bypass variants, encoding scripts, and verified-working PoC chains collected from engagements. This skill routes payload requests to the right category directory and selects appropriate variants based on target WAF, injection context, and encoding requirements.

All payloads are for **authorized penetration testing only**. Never use against systems you do not own or have explicit written permission to test.

---

## Vault Directory Structure

```
/srv/dr/payloads/
├── README.md                        # Overview and usage
├── index.json                       # Machine-readable payload catalog
├── Web/
│   ├── SQLi/
│   │   ├── MySQL/
│   │   ├── MSSQL/
│   │   ├── PostgreSQL/
│   │   ├── Oracle/
│   │   ├── SQLite/
│   │   └── Generic/
│   ├── XSS/
│   │   ├── Reflected/
│   │   ├── Stored/
│   │   ├── DOM/
│   │   ├── Polyglot/
│   │   └── WAF-Bypass/
│   ├── SSTI/
│   │   ├── Jinja2/
│   │   ├── Twig/
│   │   ├── FreeMarker/
│   │   └── Generic/
│   ├── SSRF/
│   ├── XXE/
│   ├── LFI/
│   ├── RFI/
│   ├── Command-Injection/
│   ├── Path-Traversal/
│   ├── Open-Redirect/
│   └── CSRF/
├── Auth/
│   ├── JWT/
│   ├── OAuth/
│   └── Password-Reset/
├── Deserialization/
│   ├── Java/
│   ├── PHP/
│   ├── .NET/
│   └── Python/
├── Network/
│   ├── DNS-Rebinding/
│   └── HTTP-Smuggling/
├── WAF-Bypass/
│   ├── Cloudflare/
│   ├── AWS-WAF/
│   ├── Imperva/
│   ├── ModSecurity/
│   └── Generic-Encoding/
├── Encoders/
│   ├── encode.py                    # Multi-format encoder script
│   └── decoder.py
└── Wordlists/
    ├── parameters.txt
    ├── subdomains.txt
    └── directories.txt
```

---

## Category Lookup — Quick Reference

| Request keyword | Vault path | Key files |
|---|---|---|
| SQL injection, SQLi | `Web/SQLi/<DBMS>/` | `payloads.txt`, `union.txt`, `blind.txt`, `oob.txt` |
| XSS, cross-site scripting | `Web/XSS/` | `basic.txt`, `polyglot.txt`, `waf-bypass.txt` |
| SSTI, template injection | `Web/SSTI/<engine>/` | `detect.txt`, `rce.txt` |
| SSRF | `Web/SSRF/` | `cloud-imds.txt`, `internal-ranges.txt`, `gopher.txt` |
| XXE | `Web/XXE/` | `classic.txt`, `oob.txt`, `ssrf.txt` |
| LFI, path traversal | `Web/LFI/` | `linux.txt`, `windows.txt`, `bypass.txt` |
| Command injection, RCE | `Web/Command-Injection/` | `unix.txt`, `windows.txt`, `blind.txt` |
| JWT | `Auth/JWT/` | `alg-none.txt`, `key-confusion.txt` |
| Java deserialization | `Deserialization/Java/` | `ysoserial-chains.txt` |
| PHP deserialization | `Deserialization/PHP/` | `phpggc-chains.txt` |

---

## WAF-Bypass Payload Selection

### Step 1 — Identify the WAF

```bash
# Quick fingerprint
curl -s -o /dev/null -w "%{http_code}" -H "X-Scanner: test" https://TARGET/?id=1'
wafw00f https://TARGET
```

Match output to:
- `Cloudflare` → use `WAF-Bypass/Cloudflare/`
- `AWS WAF` → use `WAF-Bypass/AWS-WAF/`
- `Imperva / Incapsula` → use `WAF-Bypass/Imperva/`
- `ModSecurity` → use `WAF-Bypass/ModSecurity/`
- Unknown / generic → use `WAF-Bypass/Generic-Encoding/`

### Step 2 — Select by injection type

For each WAF directory, payloads are organized as:
```
WAF-Bypass/<WAF>/
├── sqli.txt       # SQLi bypass payloads specific to this WAF
├── xss.txt        # XSS bypass payloads
├── rce.txt        # RCE / command injection bypass
└── notes.md       # WAF-specific quirks and detection thresholds
```

### Step 3 — Apply encoding variants

Use `/srv/dr/payloads/Encoders/encode.py`:

```bash
python3 /srv/dr/payloads/Encoders/encode.py \
  --payload "<script>alert(1)</script>" \
  --modes url,double-url,html,unicode,hex,base64

# Output example:
# url:         %3Cscript%3Ealert%281%29%3C%2Fscript%3E
# double-url:  %253Cscript%253Ealert%25281%2529%253C%252Fscript%253E
# html:        &#60;script&#62;alert&#40;1&#41;&#60;&#47;script&#62;
# unicode:     \u003cscript\u003ealert(1)\u003c/script\u003e
# hex:         \x3cscript\x3ealert(1)\x3c/script\x3e
# base64:      PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==
```

---

## Common Payload Categories — Usage Patterns

### SQLi — picking the right file

```bash
VAULT="/srv/dr/payloads/Web/SQLi"

# Generic detection (any DBMS)
cat $VAULT/Generic/detect.txt

# MSSQL-specific (xp_cmdshell, stacked queries)
cat $VAULT/MSSQL/stacked.txt
cat $VAULT/MSSQL/rce.txt

# MySQL time-based blind
cat $VAULT/MySQL/blind-time.txt

# OOB exfil (DNS)
cat $VAULT/Generic/oob-dns.txt
```

### XSS — picking by reflection context

```bash
VAULT="/srv/dr/payloads/Web/XSS"

# Basic HTML context
cat $VAULT/Reflected/html-context.txt

# JavaScript string context
cat $VAULT/Reflected/js-string-context.txt

# Attribute context
cat $VAULT/Reflected/attr-context.txt

# Polyglot (works across multiple contexts)
cat $VAULT/Polyglot/universal.txt

# Cloudflare bypass
cat $VAULT/WAF-Bypass/cloudflare-xss.txt
```

### Command injection

```bash
VAULT="/srv/dr/payloads/Web/Command-Injection"

# Unix/Linux OOB detection (need interactsh or Burp Collaborator)
cat $VAULT/unix-oob.txt

# Windows OOB detection
cat $VAULT/windows-oob.txt

# Blind time-based (no OOB)
cat $VAULT/blind-time.txt
```

---

## Updating the Payload Vault

### Sync from PayloadsAllTheThings (monthly)

```bash
cd /srv/dr/payloads
git remote add upstream https://github.com/swisskyrepo/PayloadsAllTheThings.git 2>/dev/null || true
git fetch upstream
git merge upstream/master --no-edit
# Resolve any conflicts in favour of our curated additions
git add -A && git commit -m "sync: PayloadsAllTheThings $(date +%Y-%m-%d)"
echo "Sync complete"
```

### Add a curated payload

1. Identify the correct category directory
2. Append to the relevant `.txt` file — one payload per line
3. Add a comment line above it: `# Source: <CVE or engagement ref> — <date>`
4. If it's a WAF bypass, also add to the WAF-specific directory
5. Rebuild `index.json`:

```bash
python3 /srv/dr/payloads/build-index.py
```

### index.json schema

```json
{
  "updated_at": "ISO-8601",
  "categories": [
    {
      "id": "xss-cloudflare-bypass",
      "path": "WAF-Bypass/Cloudflare/xss.txt",
      "type": "xss",
      "waf": "cloudflare",
      "count": 47,
      "last_updated": "2026-10-15",
      "notes": "Tested against CF v2.x, encoding tricks required"
    }
  ]
}
```

---

## Payload Selection Decision Tree

```
User asks for payloads
        ↓
What injection type? → look up Category Lookup table above
        ↓
Is there a WAF? → yes → identify WAF → use WAF-Bypass/<WAF>/<type>.txt
                → no  → use Web/<type>/ directly
        ↓
Are basic payloads blocked? → try encoding variants via encode.py
        ↓
Still blocked? → combine: double-encode + WAF-specific bypass + comment obfuscation
```

---

## Encoding Reference — Quick Cheat-Sheet

| Encoding | When to use | Example (`<`) |
|---|---|---|
| URL encode | GET params, query strings | `%3C` |
| Double URL encode | Proxy-decoded then re-evaluated | `%253C` |
| HTML entity | HTML attributes and body | `&#60;` or `&lt;` |
| Unicode escape | JavaScript string context | `\u003c` |
| Hex escape | JavaScript / some template engines | `\x3c` |
| Base64 + eval | When `eval(atob(...))` is possible | `PHNj...` |
| Null byte | Old PHP/C parsers | `%00` after extension |
| Newline inject | Header injection, log injection | `%0d%0a` |

---

## Related Skills

- `waf-bypass-techniques` — deep WAF fingerprint and bypass methodology
- `xss-specialist` — XSS-specific exploitation chains
- `sqli-specialist` — full SQLi attack trees per DBMS
- `rce-specialist` — RCE chain building
- `cve-tracker` — keeps payload CVE references current
