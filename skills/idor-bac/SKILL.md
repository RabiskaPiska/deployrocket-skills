---
id: idor-bac
name: IDOR and Broken Access Control Specialist
description: >
  IDOR (Insecure Direct Object Reference) and Broken Access Control specialist
  for authorized pentests and bug bounty. Covers IDOR taxonomy, parameter fuzzing,
  horizontal and vertical privilege escalation, BOLA, mass assignment, JWT claim
  manipulation, path traversal variants, and API-level access control bypass.
  Use when testing for object-level authorization flaws in web apps and APIs.
triggers:
  - IDOR
  - insecure direct object reference
  - broken access control
  - horizontal privilege escalation
  - vertical privilege escalation
  - BOLA
  - broken object level authorization
  - object reference
  - access control bypass
  - mass assignment
  - parameter tampering
  - authorization bypass
version: "1.0"
author: fleet-complete-seat-18
created: 2026-10-30
references:
  - https://owasp.org/Top10/A01_2021-Broken_Access_Control/
  - https://portswigger.net/web-security/access-control/idor
  - https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/
---

# IDOR and Broken Access Control — Specialist Playbook

## 1. Threat Model and Scope

Broken Access Control (OWASP A01:2021) is the #1 web vulnerability class.
IDOR is the most impactful sub-class: the server exposes a reference to an
internal object (DB row, file, account) and trusts the client to only request
objects it owns — without enforcing that check server-side.

**BOLA** (Broken Object Level Authorization) is the API-world synonym, ranked
#1 in OWASP API Security Top 10 2023.

Impact range: information disclosure → full account takeover → cross-tenant data
breach → privilege escalation to admin.

---

## 2. IDOR Taxonomy

### 2.1 Direct Object Reference
The raw identifier is exposed in the request.

```
GET /api/invoices/8472          ← numeric ID
GET /api/user/profile?id=8472
DELETE /orders/8472
```

Attack: increment/decrement, replace with another user's known ID.

### 2.2 Indirect Object Reference
The identifier is encoded or hashed but still predictable or guessable.

| Encoding | Example | Attack |
|---|---|---|
| Base64 | `dXNlcl9pZD04NDcy` | decode → tamper → re-encode |
| MD5/SHA1 of int | `b14a7b8059d9c055954c92674ce60032` | precompute MD5s 1-100000 |
| UUID v1 | `550e8400-e29b-41d4-a716-...` | time-based, partially predictable |
| UUID v4 | `f47ac10b-58cc-4372-a567-...` | brute-force infeasible — look for leakage instead |
| Slugs | `/users/john-doe` | enumerate via wordlist or known patterns |
| Filenames | `/uploads/receipt_84.pdf` | sequential or predictable naming |

### 2.3 Chained IDOR
One IDOR leaks an ID that unlocks a second, more sensitive IDOR.

```
Step 1: GET /api/notifications → leaks order_id values in response body
Step 2: GET /api/orders/{leaked_order_id} → full order with PII
```

Always capture all IDs returned in responses — they are stepping stones.

### 2.4 Second-Order / Stored IDOR
The reference is stored (e.g. in a webhook, email, PDF export) and resolved
later. The authorization check happens at resolution time, not at creation time.

```
POST /export { "report_id": 9999 }    ← submitted by attacker
← 202 Accepted, export queued
GET /export/download/abc123           ← victim's export URL, attacker's data
```

### 2.5 State-Changing IDOR (BFLA — Broken Function Level Authorization)
An action (PUT/DELETE/POST) is gated only on ownership at read level, not write.

```
GET  /api/documents/42   → 403 (not your doc)
PUT  /api/documents/42   → 200 (update succeeds — different code path)
DELETE /api/documents/42 → 200 (delete succeeds)
```

Always test ALL HTTP methods on endpoints that returned 403 on GET.

---

## 3. Parameter Types to Test

| Parameter type | Where found | Test approach |
|---|---|---|
| Numeric IDs | Path, query, body | ±1, ±100, 0, negative, other user's IDs |
| GUIDs/UUIDs | Path, header, cookie | Collect from responses; v1: predict; v4: harvest |
| Hashes (MD5/SHA1) | Query params, hidden fields | Precompute hash(1..N) for small ID spaces |
| Filenames | Multipart upload, download path | Predict pattern, try `../` traversal, other users' filenames |
| Email/username | Query, body | Swap to another user's identifier |
| JWT `sub`/claims | Authorization header | Decode; tamper; re-sign (see §6) |
| Array indices | `items[0]`, `line_items[2]` | Replace index, wrap to negative |
| Batch/bulk endpoints | `{ "ids": [1,2,3] }` | Inject IDs of other users |

---

## 4. Testing Methodology — Two-Account Pattern

This is the authoritative baseline for any IDOR engagement.

### 4.1 Setup
1. Register **User A** (attacker-controlled account 1).
2. Register **User B** (attacker-controlled account 2, simulates victim).
3. Use two browser profiles or two Burp Suite sessions simultaneously.
4. Create test data as User B: orders, documents, messages, profile entries.
5. Note every ID, filename, and reference that User B's resources generate.

### 4.2 Systematic Sweep
For every endpoint you identify:

```
Step 1: Perform the action as User B → capture resource ID from response.
Step 2: Switch to User A's session (cookie/token).
Step 3: Replay the identical request with User B's resource ID.
Step 4: Observe: 200 with data = confirmed IDOR; 403/404/401 = protected.
Step 5: Test all HTTP methods (GET/PUT/PATCH/DELETE/POST) on the same ID.
Step 6: Remove the Authorization header entirely → confirms whether auth check exists.
```

### 4.3 Automation with ffuf / Burp Intruder

```bash
# Enumerate numeric IDs (replace session cookie):
ffuf -u "https://target.com/api/orders/FUZZ" \
     -w <(seq 1 10000) \
     -H "Cookie: session=ATTACKER_SESSION" \
     -mc 200 -o idor_hits.json

# Filter by response size to exclude generic 200s:
ffuf ... -fs 0,12,52      ← exclude empty/error-page sizes
```

Burp Suite: send to Intruder → Sniper on the ID field → payload: number list
1-10000. Grep responses for victim email/name to confirm cross-account data.

---

## 5. Mass Assignment Vulnerabilities

The server binds ALL request body fields to the model without an allowlist,
letting attackers set fields never intended to be user-controlled.

### 5.1 Discovery
Send a PUT/PATCH/POST and add unexpected fields:

```json
PATCH /api/users/me
{
  "name": "Attacker",
  "email": "attacker@evil.com",
  "role": "admin",
  "is_verified": true,
  "credits": 99999,
  "plan": "enterprise"
}
```

If `role`, `is_verified`, or `credits` change → mass assignment confirmed.

### 5.2 Field Discovery Techniques
- Read the API documentation / Swagger spec for hidden fields.
- Fetch `GET /api/users/me` with admin account — note fields not present in
  regular user's response (those are the targets).
- Look at JavaScript source files for model field names.
- Try fields from similar endpoints (e.g. `/admin/users` response fields).

---

## 6. JWT Claim Manipulation

### 6.1 Decode and Inspect

```bash
# Decode without verification:
echo "eyJ..." | cut -d. -f2 | base64 -d | jq .
```

Target claims: `sub`, `user_id`, `role`, `admin`, `org_id`, `account_id`.

### 6.2 Algorithm Confusion (alg:none)

```bash
# Set algorithm to none, strip signature:
python3 - <<'EOF'
import base64, json
header = base64.urlsafe_b64encode(
    json.dumps({"alg":"none","typ":"JWT"}).encode()
).rstrip(b'=').decode()
payload = base64.urlsafe_b64encode(
    json.dumps({"sub":"victim_user_id","role":"admin"}).encode()
).rstrip(b'=').decode()
print(f"{header}.{payload}.")
EOF
```

### 6.3 RS256 → HS256 Key Confusion
If server uses RS256 and the public key is retrievable (JWKS endpoint, TLS cert):

```bash
# Sign with the public key as HMAC secret — server may verify HS256 with the same key:
python3 jwt_rs256_hs256_confuse.py --pub server_public.pem \
        --claim sub=victim_id --claim role=admin
```

### 6.4 Weak Secret Brute-Force

```bash
hashcat -a 0 -m 16500 target.jwt /usr/share/wordlists/rockyou.txt
# OR:
john --format=HMAC-SHA256 --wordlist=rockyou.txt jwt.txt
```

---

## 7. Path Traversal as IDOR Variant

Path traversal often achieves the same goal as IDOR: accessing a file/object
belonging to another user or system component.

```
GET /api/files/download?name=invoice_84.pdf
→ try: name=../admin/config.json
→ try: name=../../etc/passwd
→ try: name=....//....//etc/passwd  (double-slash bypass)
→ try: name=%2e%2e%2f%2e%2e%2fetc%2fpasswd  (URL encoding)
```

Combined with IDOR: if files are stored per-user in `/uploads/{user_id}/`,
test `/uploads/other_user_id/` directly.

---

## 8. API Endpoint Access Control Testing

### 8.1 Function-Level Authorization
Admin functions often share the same base URL with different path segments:

```
Regular user can access:    GET  /api/v1/users/me
Admin-only endpoints:       GET  /api/v1/admin/users
                            POST /api/v1/admin/users/create
                            GET  /api/v1/users/{any_id}   ← IDOR if unguarded
```

Attack: enumerate admin paths via wordlist (SecLists/Discovery/Web-Content/api/).

### 8.2 HTTP Method Override
Some frameworks honor `X-HTTP-Method-Override` or `_method` parameter:

```
POST /api/users/42
X-HTTP-Method-Override: DELETE
```

### 8.3 Version Downgrade
Older API versions often have relaxed access controls:

```
/api/v2/users/42    → 403
/api/v1/users/42    → 200 (v1 has no auth check on this endpoint)
/api/beta/users/42  → 200
/api/internal/users/42 → 200
```

---

## 9. Severity Rating (CVSS Guidance)

| Scenario | CVSS Base | Typical Severity |
|---|---|---|
| Read another user's PII (name, email) | 6.5–7.5 | High |
| Read financial records / PAN / SSN | 7.5–8.5 | High–Critical |
| Modify/delete another user's data | 7.5–8.5 | High–Critical |
| Delete any resource (no auth) | 8.0–9.0 | Critical |
| Escalate to admin role | 8.8–9.8 | Critical |
| Cross-tenant data breach (multi-tenant SaaS) | 9.0–10.0 | Critical |

Use https://www.first.org/cvss/calculator/3.1 — set:
- AV:N / AC:L / PR:L / UI:N for typical authenticated IDOR
- PR:N when no authentication at all is required

---

## 10. Remediation Checklist

For developers / defenders — verify each item is implemented server-side:

- [ ] **Ownership check on every object access**: confirm `object.owner_id == current_user.id`
      (or equivalent RBAC check) before returning or modifying any resource.
- [ ] **Indirect references**: map session-scoped tokens to internal IDs server-side;
      never expose raw database PKs to clients.
- [ ] **Allowlist for mass assignment**: use `@JsonProperty(access = READ_ONLY)`,
      `attr_accessible`, `[Bind]` allowlists, or DTO pattern — never bind raw request to model.
- [ ] **Consistent HTTP method enforcement**: the same access check must apply to
      GET, PUT, PATCH, DELETE, POST on the same resource.
- [ ] **API versioning**: old API versions must enforce identical access controls
      as the current version or be decommissioned.
- [ ] **JWT validation**: enforce `alg` header allowlist server-side (reject `none`);
      never accept HS256 when RS256 is configured; rotate secrets ≥256 bits.
- [ ] **Rate limiting and logging**: log all 403 responses with the requesting user ID
      — anomalous spike of 403s on sequential IDs = IDOR probing in progress.
- [ ] **Automated tests**: write integration tests that confirm User A cannot access
      User B's resources (these tests catch regressions at deploy time).

---

## 11. Tools Quick Reference

| Tool | Purpose | Install |
|---|---|---|
| Burp Suite Pro | Intruder for ID enumeration; Comparer for response diff | https://portswigger.net |
| ffuf | Fast ID fuzzing on endpoints | `go install github.com/ffuf/ffuf/v2@latest` |
| Autorize (Burp ext.) | Automatically replays requests with low-priv cookie | BApp Store |
| AuthMatrix (Burp ext.) | Multi-user authorization matrix testing | BApp Store |
| jwt_tool | JWT decode, tamper, brute-force | `pip install jwt_tool` |
| hashcat | JWT HS256 secret brute-force | `apt install hashcat` |
| ParamSpider | Discover parameter names from JS / sitemap | `pip install paramspider` |
| SecLists | Wordlists for API path enumeration | `/usr/share/seclists` |

---

## 12. Reporting Template (Abbreviated)

```
Title: IDOR — [Read/Modify/Delete] [Resource Type] of Arbitrary Users
CVSS: [score] ([vector])
Endpoint: [METHOD] /api/...
Parameter: [param name and type]
Impact: Attacker can [action] [resource] belonging to any user.

Proof of Concept:
1. Authenticate as User A (attacker).
2. Send: [exact request]
3. Observe: [response showing User B's data]

Remediation: Enforce server-side ownership check: verify resource.owner_id == session.user_id
             before returning or modifying the resource.
```
