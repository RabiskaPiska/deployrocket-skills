# WAF Fingerprinting Per Vendor

Detection and identification methodology for major Web Application Firewalls. Use at the start of any web engagement to identify defensive infrastructure before testing payloads, so you select the right bypass technique from the outset.

**Scope:** Authorized penetration tests, bug bounty (passive fingerprinting is always safe; active probing stays within scope).

---

## 1. Why Fingerprint First

Sending raw SQLi or XSS payloads without knowing the WAF leads to:
- Immediate IP block (Cloudflare, Imperva aggressive mode)
- Alert fatigue on the defender side, increasing chance of detection
- Wasted time on bypasses that don't apply to the vendor

Fingerprint → confirm vendor → select vendor-specific bypass path → test.

---

## 2. Passive Fingerprinting (Zero Noise)

### 2.1 HTTP Response Headers

Inspect headers on any normal request before sending a single payload:

```bash
curl -si https://target.com/ | grep -iE 'server|x-powered|x-cache|x-amz|cf-ray|via|x-sucuri|x-protected|x-fw|x-waf|set-cookie'
```

| Header Pattern | Vendor |
|---|---|
| `CF-RAY: *` | Cloudflare |
| `Server: cloudflare` | Cloudflare |
| `X-Cache: *AkamaiGHost*` | Akamai |
| `Server: AkamaiGHost` | Akamai |
| `X-Amzn-RequestId: *` | AWS (possibly AWS WAF) |
| `X-Amz-Cf-Id: *` | AWS CloudFront |
| `X-Sucuri-ID: *` | Sucuri |
| `X-Sucuri-Cache: *` | Sucuri |
| `Server: Imperva` | Imperva Incapsula |
| `X-CDN: Incapsula` | Imperva Incapsula |
| `X-Iinfo: *` | Imperva Incapsula |
| `X-Protected-By: Sqreen` | Sqreen |
| `Server: BarracudaWAF` | Barracuda |
| `X-Powered-By: ARR*` | Microsoft ARR (IIS) |
| `X-Fw-*` | FortiWeb |

### 2.2 Cookie Patterns (Set-Cookie Header)

```bash
curl -sc /dev/null -si https://target.com/ | grep -i 'set-cookie'
```

| Cookie Name Pattern | Vendor |
|---|---|
| `__cf_bm=*` | Cloudflare Bot Management |
| `cf_clearance=*` | Cloudflare Challenge |
| `incap_ses_*` | Imperva Incapsula |
| `visid_incap_*` | Imperva Incapsula |
| `_pk_ses.*` | Piwik/Imperva correlation |
| `ak_bmsc=*` | Akamai Bot Manager |
| `bm_sz=*` | Akamai Bot Manager |
| `_abck=*` | Akamai Bot Manager |
| `AWS_WAF_TOKEN=*` | AWS WAF Captcha |
| `sucuri_cloudproxy_*` | Sucuri |
| `FORTIWAFSID=*` | FortiWeb |

### 2.3 DNS / IP Range Fingerprinting

```bash
# Resolve the target IP
dig +short target.com

# Check ASN ownership
whois <IP> | grep -iE 'org|netname|descr'
# or
curl -s https://ipinfo.io/<IP>/org
```

| ASN / IP Range | Vendor |
|---|---|
| AS13335 (Cloudflare) | Cloudflare |
| 104.16.0.0/12 | Cloudflare |
| AS20940 (Akamai) | Akamai |
| AS16509 (Amazon) + WAF headers | AWS WAF |
| AS19551 (Incapsula) | Imperva |
| AS30148 (Sucuri) | Sucuri |

---

## 3. Active Fingerprinting (Low Noise)

Send a deliberately malformed or mildly suspicious request and analyze the response.

### 3.1 Probe Request

```bash
# Send a mild XSS probe (does not exploit, just triggers WAF response)
curl -si "https://target.com/?q=<script>alert(1)</script>" | grep -iE 'HTTP/|server|cf-ray|x-cache|set-cookie|location'

# Check response body pattern
curl -s "https://target.com/?q=<script>" | head -50
```

### 3.2 Response Code Patterns

| Code | Common Meaning |
|---|---|
| 403 | ModSecurity, Imperva, FortiWeb |
| 406 | ModSecurity default action |
| 429 | Rate-limit (Cloudflare, Akamai) |
| 503 + challenge page | Cloudflare Under Attack Mode |
| 200 + CAPTCHA body | Cloudflare, hCaptcha, AWS WAF |
| 200 + redirect JS | Imperva, Cloudflare JS challenge |
| 999 | Some Akamai deployments |

---

## 4. Per-Vendor Fingerprint Detail

### 4.1 Cloudflare

**Definitive signals:**
- `CF-RAY` header present on every response
- `Server: cloudflare` header
- `__cf_bm` cookie for bot management
- `cf_clearance` cookie after JS challenge
- Error page at `https://target.com/cdn-cgi/trace` (always returns data if behind CF)

```bash
# Definitive check
curl -si https://target.com/cdn-cgi/trace
# Returns: fl=xxx, h=target.com, ip=..., ts=..., visit_scheme=https, uag=...
# This page only exists on Cloudflare origins
```

**Error page pattern:**
```
HTTP/1.1 403 Forbidden
...
<title>Access denied | target.com used Cloudflare to restrict access</title>
```
or
```html
<div class="cf-error-details">
  <h2 class="cf-error-title">Sorry, you have been blocked</h2>
```

**Bypass references:** See `waf-bypass-techniques` skill for Cloudflare-specific bypass chains.

---

### 4.2 Akamai

**Definitive signals:**
- `X-Cache: TCP_*_HIT from a*` (AkamaiGHost in value)
- `Server: AkamaiGHost`
- `ak_bmsc`, `bm_sz`, `_abck` cookies (Akamai Bot Manager)
- Error responses include "Reference #" token

```bash
# Header probe
curl -si https://target.com/ | grep -iE 'akamai|akamaiGHost|ak_bmsc|bm_sz|_abck'

# Block response pattern
curl -si "https://target.com/?a=<script>" | grep -i 'reference'
```

**Error page pattern:**
```html
<title>Access Denied</title>
...Reference&#32;&#35;18.XXXXXXXX.XXXXXXXXXX.XXXXXXXX
```

**Timing signal:** Akamai adds ~5–15ms consistent overhead vs direct IP; extreme consistency in response time is an Akamai signal.

---

### 4.3 AWS WAF / CloudFront

**Definitive signals:**
- `X-Amzn-RequestId` header
- `X-Amz-Cf-Id` header (CloudFront distribution)
- `x-amzn-trace-id` header
- Block response body is minimal JSON or HTML with no custom branding (AWS default)

```bash
curl -si https://target.com/ | grep -iE 'x-amz|x-amzn|cloudfront'
```

**Block response pattern:**
```
HTTP/1.1 403 Forbidden
x-amzn-RequestId: abc123
...
{"message":"Forbidden"}
```
or
```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"...>
<html><head><title>ERROR: The request could not be satisfied</title>
```

**Note:** AWS WAF is configured per-rule by the operator, so rule sensitivity varies widely. Test incrementally.

---

### 4.4 ModSecurity

**Definitive signals:**
- No commercial WAF headers present
- 403 or 406 with generic Apache/Nginx server header
- Response body contains "ModSecurity" string (older/misconfigured deployments)
- `SecRuleEngine` in page source (rare, only on debug/verbose mode)

```bash
# Probe and check for ModSecurity fingerprint in body
curl -si "https://target.com/?a=1 UNION SELECT 1--" | grep -i 'modsecurity\|mod_security\|NOYB'

# Default error pages
curl -si "https://target.com/?a=<script>" | grep -iE 'HTTP/|server|content-type'
```

**Default error page patterns:**
```html
<!-- Apache + ModSecurity -->
<title>403 Forbidden</title>

<!-- Nginx + ModSecurity -->
<title>400 Bad Request</title>
<!-- body: "The plain HTTP request was sent to HTTPS port" -->

<!-- OWASP CRS default action -->
HTTP/1.1 403 Forbidden
Server: Apache/2.4.xx
<!-- no WAF-specific headers — identified by elimination -->
```

**SecRule signature detection:** Some deployments expose rule IDs in response headers or bodies:
```
Mod_Security: RuleID=941100 triggered
X-Rejected-By: ModSecurity rule 942100
```

---

### 4.5 Imperva (Incapsula)

**Definitive signals:**
- `incap_ses_<port>_<siteID>` cookie
- `visid_incap_<siteID>` cookie (persistent visitor ID)
- `X-CDN: Incapsula` header
- `X-Iinfo` header with encoded session data

```bash
curl -sI https://target.com/ | grep -iE 'incap|visid|x-cdn|x-iinfo'
```

**Block response pattern:**
```html
<html>
<head><title>Request unsuccessful. Incapsula incident ID</title></head>
<body>
  <iframe src="/_Incapsula_Resource?SWHANDSHAKEDONE=0&...
  ...Incapsula incident ID: XXXXXXXX-YYYYYYYY-ZZZZZZZZ
```

**IP range:** Imperva routes through AS19551 (Incapsula, Inc.) — verify with `whois`.

---

### 4.6 F5 BIG-IP ASM

**Definitive signals:**
- `TS` cookie prefix: `TS01xxxxxx=...`
- `Server: BigIP` or `Server: BIG-IP` header
- Blocked response references "BIG-IP" or "F5"

```bash
curl -sI https://target.com/ | grep -iE 'bigip|big-ip|TS0[0-9]'
```

**Block response pattern:**
```html
<html><head><title>Request Rejected</title></head>
<body>The requested URL was rejected. Please consult with your administrator.
Your support ID is: XXXXXXXXXXXXXXXX
```

---

### 4.7 Sucuri CloudProxy

**Definitive signals:**
- `X-Sucuri-ID` header
- `X-Sucuri-Cache` header
- Sucuri IP ranges (185.93.228.0/22, 192.88.134.0/23)

```bash
curl -sI https://target.com/ | grep -i 'sucuri'
dig +short target.com | xargs -I{} curl -s https://ipinfo.io/{}/org | grep -i sucuri
```

---

## 5. wafw00f — Automated Fingerprinting Tool

```bash
# Install
pip install wafw00f

# Basic check
wafw00f https://target.com

# Verbose (shows all detected signals)
wafw00f -v https://target.com

# Test multiple URLs from file
wafw00f -i urls.txt -o results.csv

# Force check all known WAFs
wafw00f -a https://target.com

# Output format options
wafw00f https://target.com -o json -f output.json
```

**Interpreting output:**
```
[*] Checking https://target.com
[+] The site https://target.com is behind Cloudflare (Cloudflare Inc.) WAF.
[~] Number of requests: 2
```
- `[+]` = confirmed detection
- `[~]` = possible/uncertain detection
- `[-]` = no WAF detected (may mean WAF is not triggering on wafw00f's probes)

**Caveat:** wafw00f uses a signature DB. A 2026 deployment may have updated response patterns that wafw00f doesn't know. Always cross-check with manual header inspection.

---

## 6. High-Level Bypass Pointers Per Vendor

Once fingerprinted, select the bypass strategy. Full details in the `waf-bypass-techniques` skill.

| Vendor | Primary Bypass Vector |
|---|---|
| Cloudflare | Origin IP discovery, case variation, chunked encoding, Unicode normalization |
| Akamai | HTTP/2 pseudo-header order, multipart body smuggling, path confusion |
| AWS WAF | Rule-set gaps (operator must configure), body size limits (>8KB not inspected by default) |
| ModSecurity CRS | Comment injection (`/**/`), HTTP parameter pollution, double URL encoding |
| Imperva | Whitespace variants, multiline headers, JS obfuscation in XSS |
| F5 BIG-IP ASM | Nested content-type, parameter pollution, JSON/XML encoding |

**Origin IP Exposure (bypasses all CDN-based WAFs):**
```bash
# Subdomain history (finds pre-CDN IPs)
curl -s https://crt.sh/?q=target.com | grep -oP '[\w.-]+\.target\.com' | sort -u
subfinder -d target.com -silent | httpx -ip -silent | grep -v 'cloudflare\|akamai\|imperva'

# Historical DNS
curl -s "https://securitytrails.com/domain/target.com/history/a" | grep -oP '\d+\.\d+\.\d+\.\d+'

# Direct-to-origin if found: bypass CDN WAF entirely
curl -H "Host: target.com" http://ORIGIN_IP/?a=SQLI_PAYLOAD
```

---

## 7. Detection and Defense Notes

**For defenders:** WAF fingerprinting is entirely passive on the first pass. An attacker can identify your WAF vendor by looking at headers before sending a single malicious request. Countermeasures:

- Remove or mask `Server`, `CF-RAY`, `X-Cache`, and other identifying headers where possible
- Use custom error pages that don't expose vendor-specific strings
- Enable "stealth mode" / header scrubbing where the WAF vendor offers it
- Treat WAF as one layer, not a complete defense — assume a determined attacker will identify and bypass it

---

## 8. References

- wafw00f: https://github.com/EnableSecurity/wafw00f
- Cloudflare fingerprinting: https://developers.cloudflare.com/fundamentals/reference/cdn-cgi-endpoint/
- OWASP WAF Bypass: https://owasp.org/www-community/attacks/SQL_Injection_Bypassing_WAF
- Akamai Bot Manager: https://techdocs.akamai.com/bot-manager/docs
- Imperva Incapsula detection: https://docs.imperva.com
- PayloadsAllTheThings WAF Bypass: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#waf-bypass
