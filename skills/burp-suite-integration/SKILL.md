# Burp Suite Integration — F-15 Skill Component

> Authorized penetration testing only. Confirm written scope before any active scanning.

---

## Overview

Burp Suite is the industry-standard HTTP proxy and web-application security platform. This skill covers Burp as a **programmable platform** — REST API control, headless automation, extension orchestration, and the MCP stdio bridge — not just manual point-and-click usage.

---

## 1. Editions and Capability Matrix

| Feature | Community | Professional | Enterprise |
|---|---|---|---|
| Manual proxy/intercept | ✅ | ✅ | ✅ |
| REST API (/v0.1) | ❌ | ✅ | ✅ |
| Active Scanner | ❌ | ✅ | ✅ |
| Headless mode | ❌ | ✅ | ✅ |
| Turbo Intruder | ✅ | ✅ | ✅ |
| Extensions (BApp) | ✅ | ✅ | ✅ |

All automation techniques in this skill require **Professional** unless noted.

---

## 2. Burp REST API (Pro)

Burp Pro ships a local REST API on `http://localhost:1337` by default. Authenticate with the API key from `User options → Suite → REST API`.

### 2.1 Core Endpoints

```
GET  /v0.1/target/sitemap          # Full sitemap as JSON
POST /v0.1/scan                    # Launch active/passive scan
GET  /v0.1/scan/{task_id}          # Poll scan status
GET  /v0.1/issue-definitions       # All built-in issue types
GET  /v0.1/knowledge-base/issue-definitions/{type_index}  # Details for one type
```

### 2.2 Launch a Scan via REST

```bash
# POST /v0.1/scan — minimal active scan config
curl -s -X POST http://localhost:1337/v0.1/scan \
  -H "Authorization: B61XXXXXXXXXXXXXXX" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": ["https://target.example.com/"],
    "scan_configurations": [{"name":"Crawl and Audit - Balanced","type":"NamedConfiguration"}],
    "scope": {
      "include": [{"rule": "https://target.example.com/"}],
      "type": "SimpleScope"
    }
  }'
# Returns: {"task_id": "7"}
```

### 2.3 Poll Until Complete

```python
import time, requests

API = "http://localhost:1337"
KEY = "B61XXXXX"
HEADERS = {"Authorization": KEY}

def wait_for_scan(task_id, poll_interval=15):
    while True:
        r = requests.get(f"{API}/v0.1/scan/{task_id}", headers=HEADERS)
        data = r.json()
        status = data.get("scan_status")
        print(f"  [{status}] issues={data.get('issue_counts', {})}")
        if status in ("succeeded", "failed"):
            return data
        time.sleep(poll_interval)

result = wait_for_scan("7")
```

### 2.4 Pull Sitemap Entries

```python
import requests, json

r = requests.get("http://localhost:1337/v0.1/target/sitemap",
                 headers={"Authorization": "B61XXXXX"},
                 params={"url": "https://target.example.com/"})
for entry in r.json():
    print(entry["url"], entry.get("response", {}).get("status_code"))
```

---

## 3. Headless Mode (Pro)

Run Burp without a GUI — ideal for CI/CD pipelines and VPS automation.

```bash
# Basic headless launch with project file and config
java -jar burpsuite_pro.jar \
  --project-file=/tmp/scan.burp \
  --config-file=/tmp/burp-config.json \
  --unpause-spider-and-scanner

# Generate a config file for headless use:
# In Burp GUI: Project → Save copy → Export project options as JSON
```

**burp-config.json minimum for headless scan:**

```json
{
  "project_options": {
    "connections": {
      "upstream_proxies": {"use_user_options": false}
    }
  },
  "scanner": {
    "active_scanning_enabled": true,
    "active_scanning_optimization": "minimize_false_positives"
  },
  "target": {
    "scope": {
      "advanced_mode": false,
      "include": [{"enabled": true, "scheme": "https", "host": "target.example.com", "port": "", "file": "/"}]
    }
  }
}
```

### 3.1 Headless + REST API Together

```bash
# Launch headless Burp with REST API enabled
java -jar burpsuite_pro.jar \
  --project-file=/tmp/scan.burp \
  --config-file=/tmp/burp-config.json \
  --headless.mode=true \
  --api.address=127.0.0.1 \
  --api.port=1337 \
  --api.key=MY_SECURE_KEY &

sleep 10  # wait for REST to become available

# Now control via REST
curl -H "Authorization: MY_SECURE_KEY" http://localhost:1337/v0.1/target/sitemap
```

---

## 4. Proxy Setup and CA Certificate

### 4.1 Configure Proxy Listener

- Default: `127.0.0.1:8080`
- Change: Proxy → Options → Proxy Listeners → Add/Edit
- For mobile or VM targets: bind to `0.0.0.0:8080`

### 4.2 Install Burp CA Certificate

```bash
# Download the DER cert from the running Burp instance
curl -k http://127.0.0.1:8080/cert -o burp-ca.der

# Convert to PEM (for curl/requests/system trust)
openssl x509 -inform DER -in burp-ca.der -out burp-ca.pem

# Trust system-wide on Debian/Ubuntu
sudo cp burp-ca.pem /usr/local/share/ca-certificates/burp-ca.crt
sudo update-ca-certificates

# Use with Python requests
import requests
s = requests.Session()
s.proxies = {"http": "http://127.0.0.1:8080", "https": "http://127.0.0.1:8080"}
s.verify = "/path/to/burp-ca.pem"

# Use with curl
curl --proxy http://127.0.0.1:8080 --cacert burp-ca.pem https://target.example.com/
```

---

## 5. MCP Bridge Pattern

Drive Burp programmatically via an MCP stdio server — enables AI agents to call Burp actions as tool calls.

### 5.1 Architecture

```
AI Agent (Claude / OpenCode)
        │  MCP tool call (JSON)
        ▼
  MCP stdio server (Python/Node)
        │  HTTP REST call
        ▼
  Burp Suite Pro (REST API :1337)
        │  proxy traffic
        ▼
  Target application
```

### 5.2 Minimal MCP Burp Bridge (Python)

```python
#!/usr/bin/env python3
"""burp-mcp-bridge.py — MCP stdio server that wraps Burp REST API"""
import sys, json, requests

BURP_API = "http://localhost:1337"
BURP_KEY = "B61XXXXX"
HDR = {"Authorization": BURP_KEY, "Content-Type": "application/json"}

TOOLS = [
    {"name": "burp_scan", "description": "Launch Burp active scan on a URL",
     "inputSchema": {"type":"object","properties":{"url":{"type":"string"}},"required":["url"]}},
    {"name": "burp_sitemap", "description": "Get current sitemap for a base URL",
     "inputSchema": {"type":"object","properties":{"base_url":{"type":"string"}},"required":["base_url"]}},
]

def handle(req):
    m = req.get("method")
    if m == "tools/list":
        return {"tools": TOOLS}
    if m == "tools/call":
        name = req["params"]["name"]
        args = req["params"].get("arguments", {})
        if name == "burp_scan":
            r = requests.post(f"{BURP_API}/v0.1/scan", headers=HDR,
                json={"urls":[args["url"]],"scan_configurations":[{"name":"Crawl and Audit - Balanced","type":"NamedConfiguration"}]})
            return {"content":[{"type":"text","text":r.text}]}
        if name == "burp_sitemap":
            r = requests.get(f"{BURP_API}/v0.1/target/sitemap", headers=HDR, params={"url":args["base_url"]})
            entries = [e["url"] for e in r.json()]
            return {"content":[{"type":"text","text":json.dumps(entries[:50])}]}
    return {"error": "unknown method"}

for line in sys.stdin:
    req = json.loads(line)
    resp = {"id": req.get("id"), "result": handle(req)}
    print(json.dumps(resp), flush=True)
```

```bash
# Register in opencode settings.json
# "mcpServers": {
#   "burp": {"type":"stdio","command":"python3","args":["/opt/burp-mcp-bridge.py"]}
# }
```

---

## 6. Key Extensions — Install and CLI Usage

### 6.1 Active Scan++ (Portswigger)

Extends Burp's active scanner with additional checks: SSRF, XXE, code injection, prototype pollution.

```bash
# Download from BApp Store or build from source
# https://github.com/PortSwigger/active-scan-plus-plus

# Install via Burp GUI: Extensions → BApp Store → Active Scan++
# Then runs automatically during any active scan — no config needed
```

### 6.2 Autorize — Authorization Testing

Detects broken access control by replaying requests with a lower-privilege session.

```bash
# Install: Extensions → BApp Store → Autorize
# Config:
# 1. Paste low-privilege user's Cookie header into Autorize's "Headers of low privileged user" box
# 2. Browse the app as high-privilege user → Autorize replays every request with the low-priv cookie
# 3. Red = enforced, Green = BYPASSED (broken access control finding!)
```

### 6.3 Param Miner — Hidden Parameter Discovery

Discovers hidden parameters via wordlist and header fuzzing.

```bash
# Install: Extensions → BApp Store → Param Miner

# Usage via right-click → Extensions → Param Miner → Guess params
# Or via Intruder with the miner's wordlist at:
# ~/.BurpSuite/bapps/param-miner-wordlist.txt

# Key options:
# - Fuzz params: adds random params to detect cache poisoning
# - Guess params: dictionary-based parameter discovery
# - Add custom wordlist from SecLists: /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt
```

### 6.4 JWT Editor — JWT Attack Platform

Modifies and attacks JWT tokens: alg:none, RS256→HS256 confusion, key injection.

```bash
# Install: Extensions → BApp Store → JWT Editor

# Attacks available in Repeater → JSON Web Token tab:
# - "None" Algorithm attack: change alg to "none", remove signature
# - Embedded JWK: inject a self-signed public key in the token header
# - RS256 → HS256 confusion: sign with server's public key as HMAC secret
```

### 6.5 Turbo Intruder — High-Speed Fuzzing

Sends thousands of requests per second; critical for race conditions, credential stuffing, and large wordlists.

```python
# Example: race condition attack (single-packet)
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           requestsPerConnection=100,
                           pipeline=True)
    for i in range(50):
        engine.queue(target.req, gate='race1')
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

```python
# Example: password spray — slow with lockout avoidance
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=5,
                           requestsPerConnection=1,
                           pipeline=False,
                           engine=Engine.THREADED)
    for password in open('/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt'):
        engine.queue(target.req, password.rstrip(), throttle=2000)  # 2s delay

def handleResponse(req, interesting):
    if '302' in req.response or 'dashboard' in req.response:
        table.add(req)
```

### 6.6 HTTP Request Smuggler

Detects and exploits HTTP request desync/smuggling vulnerabilities (CL.TE, TE.CL, h2.TE).

```bash
# Install: Extensions → BApp Store → HTTP Request Smuggler

# Usage: right-click any request → Extensions → HTTP Request Smuggler → Smuggle probe
# Auto-detects CL.TE / TE.CL / h2.TE variants
# Then use the Attack menu to launch differential response detection
```

---

## 7. Practical Workflows

### 7.1 Intercept Login → Replay → Fuzz → Report

```
Step 1: Setup
  - Proxy listener: 127.0.0.1:8080
  - Install CA cert in browser
  - Open target login page

Step 2: Intercept Login Request
  - Proxy → Intercept → On
  - Submit login form
  - Burp captures: POST /login with username=&password=
  - Forward or drop as needed

Step 3: Send to Repeater
  - Right-click request → Send to Repeater
  - Modify values, re-send, observe responses
  - Test: SQLi in username, default creds, CSRF bypass

Step 4: Send to Intruder / Turbo Intruder
  - Right-click → Send to Intruder
  - Mark §password§ as payload position
  - Payloads: SecLists wordlist or custom list
  - Attack type: Sniper
  - Start attack → sort by response length/status

Step 5: Analyze and Report
  - Flag anomalous responses (200 on 302, different length)
  - Confirm manually in Repeater
  - Save request/response pair
  - Generate report: Scanner → Report → HTML/XML
```

### 7.2 Full Authenticated Scan

```bash
# 1. Log in manually, capture authenticated session cookie
# 2. In Scanner settings → Session Handling Rules → Add:
#    - Rule: "Add cookie to matching requests"
#    - Scope: Scanner
#    - Cookie: session=abc123...

# 3. Or use macro: Project Options → Sessions → Macros
#    - Record login macro
#    - Tie to session handling rule to re-authenticate when session expires

# 4. Launch scan with authenticated context
# Scanner → New scan → URLs: https://target.example.com/
# Scan type: Crawl and audit
# Scope: target.example.com
```

### 7.3 Collaborator-Based Blind Detection

```bash
# Burp Collaborator detects out-of-band interactions for blind SQLi, SSRF, XXE, RCE

# In any request, insert a Burp Collaborator payload:
# Right-click → Insert Collaborator payload
# Sends a unique subdomain: xxxxxxxxxxx.oastify.com

# Monitor: Burp Collaborator client → Poll now
# DNS/HTTP/SMTP interactions from target confirm blind vulnerability
```

---

## 8. Automation Pipeline (CI/CD Integration)

```bash
#!/bin/bash
# burp-ci-scan.sh — runs Burp scan in CI, fails on High/Critical findings

TARGET="https://staging.example.com"
API_KEY="B61XXXXX"
API="http://localhost:1337"

# Wait for Burp to be ready
until curl -sf -H "Authorization: $API_KEY" "$API/v0.1/target/sitemap?url=$TARGET" > /dev/null; do
  sleep 5
done

# Launch scan
TASK_ID=$(curl -sf -X POST "$API/v0.1/scan" \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  -d "{\"urls\":[\"$TARGET\"],\"scan_configurations\":[{\"name\":\"Crawl and Audit - Fast\",\"type\":\"NamedConfiguration\"}]}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['task_id'])")

echo "Scan task: $TASK_ID"

# Poll until done
while true; do
  STATUS=$(curl -sf -H "Authorization: $API_KEY" "$API/v0.1/scan/$TASK_ID" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['scan_status'])")
  echo "Status: $STATUS"
  [[ "$STATUS" == "succeeded" || "$STATUS" == "failed" ]] && break
  sleep 30
done

# Check for High/Critical issues → fail build
HIGH=$(curl -sf -H "Authorization: $API_KEY" "$API/v0.1/scan/$TASK_ID" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('issue_counts',{}).get('high',0))")
CRITICAL=$(curl -sf -H "Authorization: $API_KEY" "$API/v0.1/scan/$TASK_ID" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('issue_counts',{}).get('critical',0))")

echo "High: $HIGH  Critical: $CRITICAL"
[ "$HIGH" -gt 0 ] || [ "$CRITICAL" -gt 0 ] && exit 1
exit 0
```

---

## 9. Detection Evasion with Burp

```
Slow scanning:
  Scanner → Options → Throttle between requests: 2000ms
  Concurrent threads: 1

Rotate source IP via upstream proxy:
  Project Options → Connections → SOCKS proxy: socks5://127.0.0.1:9050 (Tor)
  or rotate through proxy list via a local rotator on :1081

User-Agent randomization:
  Proxy → Options → Response modification → Randomize User-Agent (Extender hook)

Rate limit bypass:
  Turbo Intruder with throttle= parameter to stay under WAF rate limits
```

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| REST API returns 401 | Wrong/missing API key | Check User Options → REST API → key |
| Headless exits immediately | No project file | Always pass `--project-file` |
| CA cert not trusted | Not imported in browser | Visit http://burp → Download cert |
| Extensions not loading | Java classpath issue | Use BApp Store, not manual jar |
| Turbo Intruder 0 req/s | Pipeline off on HTTP/2 | Set `pipeline=False`, `engine=Engine.HTTP2` |

---

## References

- Burp REST API docs: https://portswigger.net/burp/documentation/desktop/tools/api
- Headless Burp: https://portswigger.net/burp/documentation/desktop/getting-started/headless
- BApp Store: https://portswigger.net/bappstore
- Turbo Intruder: https://github.com/PortSwigger/turbo-intruder
- HTTP Request Smuggler: https://github.com/PortSwigger/http-request-smuggler
