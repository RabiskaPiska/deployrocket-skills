---
name: frontend-xss-csp
description: Frontend security auditing — XSS discovery with dalfox (reflected, stored, DOM-based), CSP policy analysis and bypass patterns, CORS misconfiguration testing, Subresource Integrity enforcement, browser security headers audit (HSTS, X-Frame-Options, Permissions-Policy). Use when reviewing a web app for client-side injection vulnerabilities, auditing CSP for bypasses, checking CORS policy for credential-bearing cross-origin requests, or hardening HTTP security headers.
---

## Trigger

Fire when: user says "XSS", "CSP", "CORS", "content security policy", "cross-site scripting", "DOM injection", "security headers", "dalfox", "subresource integrity", "SRI", "clickjacking", "browser security audit", or any request to find/prevent client-side injection.

---

## Tool Decision Tree

```
Looking for XSS vulnerabilities (automated)?
  YES → dalfox url / dalfox pipe / dalfox file
Auditing CSP policy?
  YES → csp-evaluator CLI or https://csp-evaluator.withgoogle.com
Checking CORS configuration?
  YES → corsy or manual curl test patterns below
Auditing all HTTP security headers?
  YES → nuclei -t http/headers/ or securityheaders.com check
Checking script tags for missing SRI?
  YES → sri-hash CLI + manual audit below
```

---

## 1. dalfox — XSS Scanner

**Install:**
```bash
go install github.com/hahwul/dalfox/v2@latest
# or
brew install dalfox
```

### Scan a single URL
```bash
# Basic reflected XSS scan
dalfox url "https://target.com/search?q=test"

# With custom headers (authenticated scan)
dalfox url "https://target.com/search?q=test" \
  -H "Cookie: session=abc123" \
  -H "Authorization: Bearer token"

# With a custom blind XSS callback
dalfox url "https://target.com/search?q=test" \
  --blind https://your-xsshunter.com/callback

# Output as JSON
dalfox url "https://target.com/search?q=test" --format json
```

### Scan multiple URLs from file (pipe mode)
```bash
# From a file of URLs (one per line)
dalfox file urls.txt --worker 4

# From stdin pipe
cat urls.txt | dalfox pipe --worker 4

# Crawl a site and scan all params
dalfox url "https://target.com/" --deep-domxss --follow-redirects
```

### DOM XSS scanning
```bash
# Enable headless DOM XSS detection
dalfox url "https://target.com/app" --deep-domxss \
  --skip-bav --worker 2

# With specific DOM sink patterns
dalfox url "https://target.com/" \
  --deep-domxss --timeout 30
```

### WAF bypass mode
```bash
# Try encoding bypasses automatically
dalfox url "https://target.com/?q=test" --waf-evasion

# Custom payload mutations
dalfox url "https://target.com/?q=test" \
  --custom-payload /path/to/payloads.txt
```

---

## 2. CSP Analysis

### CSP Bypass Patterns

| Bypass type | Condition | Example |
|-------------|-----------|---------|
| JSONP endpoint in allowlist | `script-src` includes a domain with JSONP | `<script src="https://allowed.com/callback?cb=alert(1)">` |
| `unsafe-inline` present | Inline scripts allowed | Directly inject `<script>alert(1)</script>` |
| `unsafe-eval` present | eval allowed | `<img onerror="eval(atob('YWxlcnQoMSk='))">` |
| `data:` in `script-src` | data URIs for scripts | `<script src="data:text/javascript,alert(1)">` |
| Wildcard (`*`) subdomain | CDN subdomain abuse | Control a subdomain that is in the wildcard |
| Missing `base-uri` | Base tag injection | `<base href="https://attacker.com/">` |
| Missing `object-src` | Flash/plugin injection | `<object data="...">` |
| `nonce` reuse | Server-side nonce is static/predictable | Steal nonce from page and reuse |

### Manual CSP audit
```bash
# Extract CSP header
curl -s -I https://target.com | grep -i content-security-policy

# Use Google CSP Evaluator CLI
npm install -g @google/csp-evaluator
csp-evaluator --url https://target.com
```

### Strong CSP template
```
Content-Security-Policy:
  default-src 'none';
  script-src 'nonce-{RANDOM}' 'strict-dynamic' https: 'unsafe-inline';
  style-src 'nonce-{RANDOM}' https:;
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  object-src 'none';
  upgrade-insecure-requests;
```

---

## 3. CORS Misconfiguration Testing

### Manual test patterns
```bash
# Check if arbitrary origin is reflected
curl -s -I -H "Origin: https://attacker.com" https://target.com/api/user \
  | grep -i "access-control-allow-origin"
# VULNERABLE if response is: Access-Control-Allow-Origin: https://attacker.com

# Check if null origin is allowed
curl -s -I -H "Origin: null" https://target.com/api/user \
  | grep -i "access-control"

# Check with credentials
curl -s -I -H "Origin: https://attacker.com" \
  -H "Cookie: session=abc" \
  https://target.com/api/sensitive \
  | grep -E "access-control-allow-(origin|credentials)"
# CRITICAL if both: ACAO: attacker.com + ACAC: true
```

### Automated CORS scan with corsy
```bash
pip install corsy
python3 corsy.py -u https://target.com/api/ -t 10
```

### CORS misconfiguration severity matrix

| Condition | Severity | Impact |
|-----------|----------|--------|
| Arbitrary origin reflected + `Allow-Credentials: true` | CRITICAL | Full cross-origin auth bypass |
| Arbitrary origin reflected, no credentials | MEDIUM | Read non-sensitive responses |
| `null` origin allowed + `Allow-Credentials: true` | HIGH | Exploitable via sandboxed iframe |
| Subdomain wildcard + `Allow-Credentials: true` | HIGH | Exploitable via XSS on any subdomain |
| Pre-flight cached too long (`Access-Control-Max-Age`) | LOW | Cache poisoning risk |

---

## 4. HTTP Security Headers Audit

### Check all headers with nuclei
```bash
nuclei -t http/headers/ -target https://target.com
```

### Manual curl audit
```bash
curl -s -I https://target.com | grep -Ei \
  "strict-transport|x-frame|x-content-type|content-security|permissions-policy|cross-origin"
```

### Required headers checklist

| Header | Required value | Blocks |
|--------|---------------|--------|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | HTTPS downgrade |
| `X-Frame-Options` | `DENY` or CSP `frame-ancestors 'none'` | Clickjacking |
| `X-Content-Type-Options` | `nosniff` | MIME-type sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Referrer leakage |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=()` | Feature abuse |
| `Cross-Origin-Opener-Policy` | `same-origin` | Spectre/XS-Leaks |
| `Cross-Origin-Embedder-Policy` | `require-corp` | Spectre/XS-Leaks |
| `Cross-Origin-Resource-Policy` | `same-origin` | Cross-origin data leaks |

---

## 5. Subresource Integrity (SRI)

### Generate SRI hashes
```bash
# Single file
curl -s https://cdn.example.com/lib.js | openssl dgst -sha384 -binary | openssl base64 -A

# Using sri-hash CLI
npm install -g sri-hash
sri-hash https://cdn.example.com/lib.js

# Using sri-toolbox
npx sri-toolbox https://cdn.example.com/lib.js
```

### Correct SRI tag format
```html
<script
  src="https://cdn.example.com/lib.min.js"
  integrity="sha384-ABC123...=="
  crossorigin="anonymous">
</script>
```

### Audit existing page for missing SRI
```bash
# Find all external scripts/styles without integrity attribute
curl -s https://target.com/ | grep -Ei '<script|<link' | grep -iv integrity
```

---

## 6. BitterSecurity/Decepticon — Automated Header Evasion

`BitterSecurity/Decepticon` (formerly `PurpleAILAB/Decepticon`) is a WAF/header-evasion toolkit.

```bash
git clone https://github.com/BitterSecurity/Decepticon
cd Decepticon
pip install -r requirements.txt

# Test header-based evasion against a target
python3 decepticon.py -u https://target.com/vulnerable?param=test \
  --technique header-mangling

# Combine with dalfox payloads
python3 decepticon.py -u https://target.com/ --xss --dalfox-integration
```

---

## DOM XSS Sink Reference

High-risk sinks to audit in source code:
```javascript
// CRITICAL SINKS (any user-controlled input → IMMEDIATE XSS)
element.innerHTML = userInput
element.outerHTML = userInput
document.write(userInput)
document.writeln(userInput)
location.href = userInput          // javascript: URI
location.assign(userInput)
location.replace(userInput)
eval(userInput)
setTimeout(userInput, 0)           // string form
setInterval(userInput, 0)
new Function(userInput)

// HIGH RISK (context-dependent)
element.insertAdjacentHTML('beforeend', userInput)
$.html(userInput)                  // jQuery
$(userInput)                       // jQuery selector injection
```

---

## References

- `hahwul/dalfox` — https://github.com/hahwul/dalfox (3.9k+ stars, MIT)
- `BitterSecurity/Decepticon` (formerly PurpleAILAB/Decepticon) — WAF evasion toolkit
- `s0md3v/Corsy` — https://github.com/s0md3v/Corsy (MIT) — CORS misconfiguration scanner
- `projectdiscovery/nuclei-templates` — http/headers/ templates (MIT)
- Google CSP Evaluator — https://csp-evaluator.withgoogle.com
- OWASP XSS Prevention Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- Source reports: `PENTEST-AI-AGENT-SKILL-MINE-2026-10-26.md`, crew-3 frontend reports