---
id: burp-extension-toolkit
name: burp-extension-toolkit
description: >
  Burp Suite extension toolkit for penetration testing. Covers essential extensions
  (Active Scan++, Autorize, Param Miner, JWT Editor, Turbo Intruder, HTTP Request
  Smuggler, Nuclei Burp Plugin), workflows for each extension, BAPP store automation,
  and headless Burp API usage for CI/CD integration. Triggers on: Burp Suite extension,
  Burp plugin, Autorize, Param Miner, JWT Editor, Turbo Intruder, HTTP Request Smuggler,
  Active Scan++, Burp BAPP, headless Burp.
triggers:
  - Burp Suite extension
  - Burp plugin
  - Autorize
  - Param Miner
  - JWT Editor
  - Turbo Intruder
  - HTTP Request Smuggler
  - Active Scan++
  - Burp BAPP
  - headless Burp
  - Burp collaborator
  - Burp intruder
---

# Burp Suite Extension Toolkit

## Setup

All extensions referenced are available free from the BApp Store (Extender → BApp Store)
unless marked [Pro] (requires Burp Suite Professional). Load order matters — install
extensions before starting a pentest engagement.

---

## 1. Active Scan++ [Pro]

**Purpose:** Extends Burp's active scanner with extra detection for SSRF, XXE, SSTI,
RCE, header injection, blind XSS, log4shell, and more.

### Installation
```
Extender → BApp Store → Search "Active Scan++" → Install
```

### Configuration and Workflow

```
1. Enable in Scanner settings: Scan type → select "Audit checks — extensions only"
   OR append Active Scan++ checks to existing audit
2. Right-click a request in Proxy History → Scan → Issue activity checks
3. Key additions Active Scan++ provides over built-in scanner:
   - Out-of-band SSRF detection via Burp Collaborator
   - SSTI detection (Jinja2, Twig, FreeMarker probes)
   - Log4Shell / Log4j CVE-2021-44228 header injection
   - Host header injection → SSRF
   - HTTP response splitting
   - SQLi in JSON bodies (Burp misses these by default)
   - Blind code injection (executes code, checks OOB callback)
```

### Custom Insertion Points (Critical for JSON APIs)

```
1. Highlight a value in a JSON request
2. Right-click → Scan → Use custom insertion point
3. Active Scan++ will probe the highlighted value with all its checks
4. Essential for: {"id": SCAN_HERE}, {"user": {"name": SCAN_HERE}}
```

---

## 2. Autorize — Broken Access Control Testing

**Purpose:** Automatically tests every request with a lower-privileged auth token to
detect IDOR and broken access control. Best extension for access control testing.

### Installation
```
Extender → BApp Store → Search "Autorize" → Install
```

### Workflow — Admin vs User Test

```
1. Log in as ADMIN user — capture session cookie/token
2. Log in as REGULAR USER in a second browser — capture their cookie/token
3. In Burp → Autorize tab:
   a. "Auth Header to Replace" — paste the regular user's cookie/token
   b. Click "Autorize is off" to toggle ON
4. Browse all admin functionality AS ADMIN (Autorize replays each request as regular user)
5. Autorize marks each request:
   BYPASSED (red)    — Regular user got same response as admin → ACCESS CONTROL BUG
   ENFORCED (green)  — Regular user got different/blocked response → OK
   IS ENFORCED? (yellow) — Need manual review
6. Export results: Autorize → Save
```

### Advanced Configuration

```
Interception Filters:
- Only intercept requests to specific paths: add URL filter (/admin, /api/v1)
- Ignore static resources: Filter → "Ignore file extensions: css,js,png,jpg"

Multiple User Roles:
- Add a second token for a SECOND_USER_ROLE
- Now tests: Admin → User1 → User2 (three-way comparison)

Unauthenticated Check:
- Add empty string as "Auth Header" → tests if unauthenticated access works
```

---

## 3. Param Miner — Hidden Parameter Discovery

**Purpose:** Discovers hidden GET/POST parameters, HTTP headers, and path parameters
by brute-forcing against a target endpoint with a massive wordlist.

### Installation
```
Extender → BApp Store → Search "Param Miner" → Install
```

### Workflow

```bash
# GUI workflow:
1. Right-click any request in Proxy History
2. Extensions → Param Miner → Guess (GET/POST/headers/cookies/path)
3. Options:
   - "Guess headers" — finds reflected/injected headers
   - "Guess params" — finds query string / POST body parameters
   - "Add FCBUs" — bruteforce cookie names
   - Check "Use Burp Collaborator" for blind parameter discovery
4. Results appear in the "Param Miner" tab under Output
5. Interesting outputs:
   - debug=true → enables debug output
   - test=1, admin=1 → toggles features
   - callback=URL → SSRF vector
   - _method=PUT → HTTP method override
```

### CLI-style via Burp REST API

```bash
# Automate param miner via Burp's REST API (Pro)
curl -s -X POST http://127.0.0.1:1337/v0.1/scan \
  -H "Content-Type: application/json" \
  -d '{
    "urls": ["https://TARGET/api/endpoint"],
    "scan_configurations": [{"type": "NamedConfiguration", "name": "Audit checks - extensions only"}]
  }'
```

### High-Value Parameters to Look For

```
debug, test, admin, internal, secret, key, token, callback, redirect,
next, url, src, source, dest, destination, file, path, include, template,
format, output, type, action, cmd, command, exec, run, shell, ping, host,
api_key, access_token, bearer, _token, csrf, nonce, session, sid
```

---

## 4. JWT Editor — JSON Web Token Attacks

**Purpose:** Modify, resign, and attack JWTs directly in Burp. Covers alg:none, RS256→HS256
confusion, weak secret brute-force, JWK injection, and JWKS endpoint spoofing.

### Installation
```
Extender → BApp Store → Search "JWT Editor" → Install
```

### Workflow — Key Attacks

#### Attack 1: alg:none (No Signature Verification)

```
1. Intercept a request with a JWT in Authorization header
2. Go to JSON Web Tokens tab in request editor
3. Header section: Change "alg": "HS256" to "alg": "none"
4. Payload: Modify claims (e.g., "role": "admin", "sub": "admin_user_id")
5. Click "Sign" → select "Don't sign" (empty signature)
6. Forward the modified request
7. Check if server accepts it → alg:none vulnerability
```

#### Attack 2: RS256 → HS256 Confusion (Public Key as HMAC Secret)

```
1. Obtain the server's RSA public key from:
   - /jwks.json
   - /.well-known/openid-configuration
   - SSL certificate public key
2. In JWT Editor → Keys tab → New RSA Key → Paste public key PEM
3. In the JWT tab: Change header alg from "RS256" to "HS256"
4. Modify payload claims
5. Sign with the imported RSA public key (now used as HMAC secret)
6. Server (incorrectly) uses public key to verify HS256 HMAC → accepts forged JWT
```

#### Attack 3: Weak HMAC Secret Brute Force

```bash
# Extract JWT from traffic (eyJ... token)
JWT="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.SIGNATURE"

# Hashcat brute force
echo "$JWT" > jwt.txt
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt

# john
john --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt jwt.txt
```

#### Attack 4: JWK Header Injection

```
1. In JWT Editor → Keys → New RSA Key → Generate
2. Copy the public key JWK
3. In JWT tab, add to header: "jwk": {PUBLIC_KEY_JWK}
4. Sign with the corresponding private key
5. If server uses the embedded JWK to verify → forged JWT accepted
```

---

## 5. Turbo Intruder — High-Speed Request Engine

**Purpose:** Python-scriptable Burp extension that sends requests at high speed.
Essential for race conditions, password brute force, and OTP bypass.

### Installation
```
Extender → BApp Store → Search "Turbo Intruder" → Install
```

### Race Condition — Single-Packet Attack (Portswigger Method 2023)

```python
# Right-click a request → Extensions → Turbo Intruder → Send to Turbo Intruder
# Use this script for single-packet race condition (HTTP/2)

def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2)  # HTTP/2 single-packet
    
    # Queue all requests to be sent in a single packet
    for i in range(20):
        engine.queue(target.req, gate='race1')
    
    # Fire all queued requests simultaneously
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

### Password Brute Force at Scale

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=5,
                           requestsPerConnection=100,
                           pipeline=False)
    
    for password in open('/usr/share/wordlists/rockyou.txt'):
        engine.queue(target.req, password.rstrip())

def handleResponse(req, interesting):
    if req.status == 302 or 'Welcome' in req.response:
        table.add(req)
```

### OTP / Rate Limit Bypass

```python
# Queue multiple OTP guesses before the server can apply rate limiting
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2)
    
    # All 4-digit PINs 0000-9999
    for code in range(10000):
        engine.queue(target.req, str(code).zfill(4), gate='otp_gate')
    
    engine.openGate('otp_gate')

def handleResponse(req, interesting):
    if '200' in req.status and 'invalid' not in req.response.lower():
        table.add(req)
```

---

## 6. HTTP Request Smuggler — Desync Detection

**Purpose:** Detects HTTP request smuggling (CL.TE, TE.CL, TE.TE, h2.CL, h2.TE)
automatically via timing and differential response techniques.

### Installation
```
Extender → BApp Store → Search "HTTP Request Smuggler" → Install
```

### Workflow

```
1. Right-click a request to the target's main endpoint
2. Extensions → HTTP Request Smuggler → Smuggle probe
3. Select probes to run:
   - CL.TE — Content-Length + Transfer-Encoding desync
   - TE.CL — Transfer-Encoding + Content-Length desync
   - TE.TE — TE obfuscation variants
   - h2.CL — HTTP/2 with Content-Length header (downgrade attack)
   - h2.TE — HTTP/2 with Transfer-Encoding header
4. Let it run (takes 30-60 seconds per endpoint)
5. Findings appear in the Scanner tab
6. False positive check: repeat the same probe 3x — consistent timing = real finding
```

### Manual Verification After Detection

```
CL.TE confirmation:
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G

(If next request gets a 403/404 starting with 'G', smuggling confirmed)
```

---

## 7. Nuclei Burp Plugin

**Purpose:** Run Nuclei templates directly against requests in Burp Proxy History.
Combines Burp's interception with Nuclei's CVE coverage.

### Installation
```
Extender → BApp Store → Search "Nuclei" → Install
# Note: Requires Nuclei to be installed and in PATH
```

### Workflow

```bash
# Install nuclei first
go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
nuclei -update-templates

# In Burp:
1. Browse the target normally — capture traffic in Proxy
2. Select requests in Proxy History → right-click → Nuclei Scan
3. Select template categories:
   - cves/          (specific CVE checks)
   - vulnerabilities/  (general vulns)
   - exposures/     (sensitive file exposure)
   - misconfiguration/  (server misconfigs)
4. Results appear in Nuclei tab with findings and severity
```

---

## 8. Other Essential Extensions

### 8.1 Retire.js — JavaScript CVE Detection

```
BApp Store → "Retire.js"
Passive scan — automatically flags outdated/vulnerable JavaScript libraries
in every response. No config needed.
```

### 8.2 Hackvertor — Encoding/Decoding

```
BApp Store → "Hackvertor"
Use for complex encoding chains: base64(url(html(input)))
Right-click text in request → Extensions → Hackvertor → Encode/Decode

Useful for:
- WAF bypass payload encoding
- JWT header/payload manipulation
- Unicode/hex encoding for injection
```

### 8.3 Logger++ — Advanced Logging and Grep

```
BApp Store → "Logger++"
Replaces Burp's proxy history with:
- Advanced filtering (regex, multiple conditions)
- Grep tab — search all traffic for specific patterns
  e.g., filter for responses containing "password" or "api_key"
- Export to CSV/JSON
- Color-coded entries by response code
```

### 8.4 Freddy — Deserialization Scanner [Pro]

```
BApp Store → "Freddy, Deserialization Bug Finder"
Detects Java, .NET, PHP, Python deserialization in requests/responses
Marks potential deserialization vectors for manual testing
Works passively and with active scanning
```

### 8.5 InQL — GraphQL Introspection and Testing

```
BApp Store → "InQL"
- Auto-introspects GraphQL schemas
- Generates test queries for all types/fields
- Identifies batch query abuse opportunities
- Detects IDOR via GraphQL IDs
```

---

## 9. Headless Burp REST API (CI/CD Integration)

```bash
# Start Burp headless with REST API enabled
java -jar burpsuite_pro.jar --disable-extensions \
  --config-file=project-config.json \
  --user-config-file=user-config.json

# Burp REST API (default port 1337)
# Enable: Project → Project options → Misc → REST API service

# Start a scan via API
curl -s -X POST "http://127.0.0.1:1337/v0.1/scan" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": ["https://TARGET"],
    "scan_configurations": [{
      "type": "NamedConfiguration",
      "name": "Audit checks - all except time-based detection methods"
    }],
    "application_logins": [{
      "username": "USER",
      "password": "PASS"
    }]
  }' | jq '.task_id'

# Check scan status
TASK_ID=1
curl -s "http://127.0.0.1:1337/v0.1/scan/$TASK_ID" | jq '.scan_status'

# Get issues (when scan_status = "succeeded")
curl -s "http://127.0.0.1:1337/v0.1/scan/$TASK_ID/issues" | \
  jq '.issues[] | {severity, name, path}'

# Generate report
curl -s -X GET "http://127.0.0.1:1337/v0.1/report" \
  -H "Content-Type: application/json" \
  -d '{"reportType": "HTML", "taskId": '$TASK_ID'}' > burp_report.html
```

---

## 10. Proxy Configuration for Tools Integration

```bash
# Route tool traffic through Burp for inspection
# Burp default: 127.0.0.1:8080

# curl
curl -x http://127.0.0.1:8080 -k https://TARGET/api

# sqlmap
sqlmap -u "https://TARGET" --proxy=http://127.0.0.1:8080

# nuclei
nuclei -u https://TARGET -proxy http://127.0.0.1:8080

# ffuf
ffuf -w wordlist.txt -u https://TARGET/FUZZ -x http://127.0.0.1:8080

# Python requests
import requests
proxies = {"https": "http://127.0.0.1:8080", "http": "http://127.0.0.1:8080"}
r = requests.get("https://TARGET", proxies=proxies, verify=False)

# Trust Burp CA for HTTPS inspection
# Export: Proxy → Options → Import/Export CA certificate
# Install in system trust store or pass with --cacert flag
```

---

## Related Skills

- `http-request-smuggling` — deep dive on smuggling techniques detected by HTTP Request Smuggler
- `race-condition` — Turbo Intruder workflows for race conditions
- `xss-specialist` — JWT Editor and Param Miner for XSS discovery
- `sqli-specialist` — Active Scan++ for SQL injection discovery
- `pentest` — overall engagement methodology
