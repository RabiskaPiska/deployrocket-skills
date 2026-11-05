---
id: oauth-oidc-attacks
name: oauth-oidc-attacks
description: >
  OAuth 2.0 and OpenID Connect attack specialist for authorized penetration tests and bug bounty.
  Covers the full OAuth/OIDC attack taxonomy: CSRF via missing state parameter, open redirect
  abuse in redirect_uri, authorization code injection, PKCE downgrade/bypass, OIDC nonce replay,
  ID token validation bypass, implicit flow token theft, refresh token abuse, and JWT-based OAuth
  token attacks (alg:none, RS256→HS256 key confusion). Includes testing methodology with Burp Suite
  OAuth scanner, manual state-parameter testing, and a structured finding schema with CVSS guidance.
  Use when testing an OAuth 2.0 / OIDC implementation, reviewing an authorization server, or
  performing a bug bounty on any login/SSO flow.
triggers:
  - oauth
  - openid
  - oidc
  - authorization code
  - state parameter
  - redirect_uri
  - token endpoint
  - PKCE
  - implicit flow
  - SSO attack
  - oauth csrf
  - token leakage
  - authorization code injection
  - jwt alg none
  - key confusion
version: "1.0"
verified: "2026-10-30"
---

# OAuth 2.0 / OIDC Attack Specialist

> **Scope gate:** every technique below requires an authorized engagement (pentest scope, bug-bounty program, or lab environment). Never test production systems without written permission.

---

## 1. Threat Model and Attack Surface Map

```
Authorization Server (AS)
  ├── /authorize endpoint          ← state, redirect_uri, PKCE, scope
  ├── /token endpoint              ← code exchange, client auth, refresh
  ├── /userinfo endpoint           ← OIDC subject claims
  └── /.well-known/openid-config   ← discovery, JWKS URI

Client Application
  ├── redirect_uri registration    ← open redirect, prefix match abuse
  ├── state / nonce generation     ← CSRF, replay
  ├── token storage                ← localStorage, sessionStorage, cookie
  └── JWT validation logic         ← alg, iss, aud, exp, nonce checks
```

**High-value targets ranked by severity:**
1. Missing or reused `state` → CSRF account takeover
2. Open redirect in `redirect_uri` → authorization code / token theft
3. Authorization code injection → token issued to attacker
4. PKCE absent or bypassable → code interception
5. Implicit flow token in URL fragment → Referer/log leakage
6. JWT `alg:none` or RS256→HS256 confusion → forged tokens
7. Refresh token lifetime/rotation abuse → persistent access

---

## 2. Attack Taxonomy

### 2.1 CSRF via Missing `state` Parameter

**Mechanism:** OAuth requires the client to generate a random `state`, bind it to the session, send it in `/authorize`, and verify it in the callback. If the client omits or skips verification, an attacker can craft a `/authorize` URL with their own account and socially-engineer the victim into completing the flow, linking the attacker's identity to the victim's account.

**Test procedure:**
```
1. Start a login-with-OAuth flow; intercept the /authorize redirect.
2. Copy the URL. Verify state is present.
3. Drop the flow. Open the callback URL (with attacker's code) in a logged-in victim session.
4. If the victim's session is now linked to the attacker's external account → VULNERABLE.
```

**Tooling:**
- Burp Suite → Proxy → intercept `/authorize` request; replace `state` or remove it entirely.
- Manual: craft a `<img src="...">` or `<iframe>` with the poisoned callback URL.

**Detection signal:** no `state` in `/authorize` GET, or `state` present but callback handler never reads/validates it (check JS source / server logs for `state` comparison).

**Hardening:** generate `state` = `HMAC(session_id, random_nonce)`; validate exact match on callback; bind to session cookie; use `SameSite=Strict` on session cookies as defense-in-depth.

---

### 2.2 Open Redirect in `redirect_uri`

**Mechanism:** Authorization servers must validate `redirect_uri` against a pre-registered allowlist. Weaknesses: prefix matching (`https://legit.com.evil.com` passes a prefix check on `https://legit.com`), path traversal (`https://legit.com/callback/../attacker`), fragment injection, and wildcard registrations (`https://legit.com/*`). A valid `redirect_uri` pointing to an attacker-controlled page receives the authorization code or access token in the URL.

**Attack variants:**
- **Subdomain takeover redirect:** `redirect_uri=https://sub.legit.com/callback` where `sub` is unclaimed.
- **Open redirect chain:** `redirect_uri=https://legit.com/redirect?to=https://evil.com`; AS validates the first hop.
- **Fragment injection:** `redirect_uri=https://legit.com/callback#attacker-fragment`.

**Test procedure:**
```bash
# Basic — append extra path
redirect_uri=https://legit.com/callback/../evil

# Subdomain confusion
redirect_uri=https://legit.com.evil.com/callback

# Open redirect chain — find an open redirect on the registered domain first
redirect_uri=https://legit.com/redirect?url=https://attacker.com
```

**Tooling:**
- Burp Intruder with a `redirect_uri` wordlist of bypass patterns (path traversal, sub-TLD, encoded slashes).
- `oauth2-proxy` test harness.
- Manually browse `/.well-known/openid-configuration` → `registration_endpoint` → inspect registered URIs.

**Detection signal:** AS returns `invalid_redirect_uri` only on obviously wrong URIs but passes prefix/wildcard variants.

**Hardening:** exact-string match only; no wildcards; reject URL-encoded slashes; register the full callback path.

---

### 2.3 Token Leakage via Referer and Logs

**Mechanism:** In the implicit flow (deprecated but still live) and some hybrid flows, the access token or authorization code appears in the URL fragment or query string. A `<script src="https://cdn.example.com">` on the callback page sends the full URL (including fragment in some browsers) as the `Referer` header to the CDN, leaking the token to third parties.

**Affected flows:** implicit (`response_type=token`), hybrid (`response_type=code token`), any flow that puts sensitive values in the query string.

**Test procedure:**
```
1. Intercept OAuth callback; look for access_token or code in URL query/fragment.
2. Check Referer header on subsequent resource loads (JS, images, fonts).
3. Check server access logs for GET requests containing token= or access_token=.
4. Check browser history, bookmarks.
```

**Hardening:** use authorization code flow + PKCE; never use implicit flow; redirect to a clean URL after consuming the code; set `Referrer-Policy: no-referrer` on the callback page.

---

### 2.4 Authorization Code Injection

**Mechanism:** If the client does not bind the `state`/`nonce` to the specific redirect interaction, an attacker who steals a valid authorization code (via open redirect, log exposure, or MITM on HTTP) can inject it into their own callback session and exchange it for tokens associated with the victim.

**PKCE (Proof Key for Code Exchange)** mitigates this: the client generates `code_verifier` (random), sends `code_challenge = SHA256(code_verifier)` in `/authorize`, and presents `code_verifier` on `/token`. An injected code fails because the attacker does not have the original `code_verifier`.

**Test for injection:**
```
1. Obtain a valid authorization code (via Burp intercept on the attacker's own OAuth flow).
2. Before exchanging it in your session, inject it into a different session's callback URL.
3. Observe whether the second session gets tokens.
```

**Hardening:** PKCE S256 mandatory for all public clients; AS must reject codes presented with mismatched `code_verifier`; codes must be single-use and short-lived (≤10 min).

---

### 2.5 PKCE Downgrade / Bypass

**Mechanism:** AS does not enforce PKCE when the client omits `code_challenge`. If PKCE use is optional, an attacker-controlled client can initiate a flow without PKCE, making the resulting code vulnerable to injection.

**Test procedure:**
```bash
# Remove code_challenge and code_challenge_method from /authorize
GET /authorize?response_type=code&client_id=CLIENT&redirect_uri=...&scope=openid
# (no code_challenge parameter)

# If AS returns a code, test whether /token accepts it without code_verifier
POST /token
  grant_type=authorization_code&code=CODE&redirect_uri=...&client_id=CLIENT
  # (no code_verifier)
```

**Detection signal:** `/authorize` succeeds without `code_challenge`; `/token` succeeds without `code_verifier`.

**Hardening:** AS must require PKCE for all public clients; PKCE should be enforced for confidential clients too per OAuth 2.1; reject `code_challenge_method=plain` (S256 only).

---

### 2.6 OIDC Nonce Replay

**Mechanism:** The OIDC `nonce` ties an ID token to a specific authentication request. If the client does not validate `nonce` in the ID token, an attacker who captures an ID token (e.g., from a log) can replay it in a new session.

**Test:**
```
1. Complete an OIDC login; capture the ID token (Burp intercept).
2. In a new session, manually supply the same ID token to the client's token-handling endpoint.
3. If the session is authenticated → nonce not validated.
```

**Hardening:** client MUST generate a unique nonce per request, store it server-side, and verify it matches the `nonce` claim in the ID token exactly once.

---

### 2.7 ID Token Validation Bypass

**Critical checks the client MUST perform (RFC 7519 + OIDC Core §3.1.3.7):**

| Claim | Required validation |
|-------|---------------------|
| `iss` | Must equal expected issuer URL |
| `aud` | Must contain the client's `client_id` |
| `exp` | Must be in the future |
| `iat` | Must be recent (clock skew ≤5 min) |
| `nonce` | Must match the session nonce |
| `alg` | Must be an expected algorithm; reject `none` |
| Signature | Verify against JWKS from `/.well-known/jwks.json` |

**Common bypass patterns:**
- **Algorithm `none`:** remove the signature, set `"alg":"none"` in header.
- **`aud` not checked:** forge token for a different audience.
- **Expired token accepted:** manipulate `exp` to past value; server never rejects.
- **`iss` not checked:** submit a token issued by attacker-controlled AS.

**Test for `alg:none`:**
```python
import base64, json

header = base64.urlsafe_b64encode(json.dumps({"alg":"none","typ":"JWT"}).encode()).rstrip(b'=')
payload = base64.urlsafe_b64encode(json.dumps({
    "sub":"victim_user_id","iss":"https://real-as.com","aud":"client_id",
    "exp":9999999999,"iat":1700000000,"nonce":"any"
}).encode()).rstrip(b'=')
forged_token = f"{header.decode()}.{payload.decode()}."
```

---

### 2.8 Implicit Flow Token Theft

**Mechanism:** `response_type=token` or `response_type=id_token` returns tokens in the URL fragment. If the SPA makes external resource requests before consuming the fragment, the fragment may leak. Additionally, `postMessage` misconfiguration in SPAs can allow cross-origin frame to receive tokens.

**Test for `postMessage` leakage:**
```javascript
// Open the SPA in an iframe, listen for postMessage events
window.addEventListener('message', e => console.log(e.origin, e.data));
```

**Hardening:** migrate to authorization code + PKCE; deprecate implicit flow; use `response_mode=fragment` only if unavoidable, and consume tokens immediately without external resource loads.

---

### 2.9 Refresh Token Abuse

**Mechanism:** If refresh tokens are long-lived, not rotated, and not bound to a specific client/device, a stolen refresh token grants persistent access even after the access token expires.

**Test:**
```
1. Capture a refresh token (Burp intercept, local storage dump).
2. Use it from a different IP/user-agent.
3. If token exchange succeeds → refresh token not bound to client context.
4. Use a rotated refresh token (the old one): if exchange still succeeds → rotation not enforced.
```

**Hardening:** rotate refresh tokens on every use; revoke on rotation (old tokens become invalid); bind to client IP/device fingerprint where feasible; enforce absolute lifetime (e.g., 30 days); store as `HttpOnly Secure SameSite=Strict` cookie.

---

### 2.10 JWT-Based OAuth Tokens — `alg:none` and RS256→HS256 Key Confusion

**`alg:none` (CVE class):**
Libraries that trust the `alg` header can be tricked into accepting an unsigned token. See §2.7 for the forge procedure.

**RS256→HS256 key confusion:**
If the server signs tokens with an RSA private key (RS256) but the library also supports HS256, an attacker can:
1. Retrieve the RS256 **public key** from `/jwks.json`.
2. Sign a forged token with HS256 using the public key as the HMAC secret.
3. Send the forged token with `"alg":"HS256"`.
4. A vulnerable library verifies the HS256 signature using its configured RS256 public key → verification passes.

```bash
# Using jwt_tool
python3 jwt_tool.py <TOKEN> -X k -pk public_key.pem
```

**Tooling:**
- `jwt_tool` (ticarpi/jwt_tool) — alg:none, key confusion, JWKS injection, claim tampering
- `python-jwt` library for manual forgery
- Burp JWT Editor extension

**Detection signal:** server returns 200 with forged token; claims in the token belong to a different user.

**Hardening:** pin the algorithm server-side; never derive it from the token header; use a vetted JWT library with explicit algorithm allowlisting.

---

## 3. Testing Methodology

### 3.1 Reconnaissance
```bash
# Discover OIDC configuration
curl https://TARGET/.well-known/openid-configuration | jq .

# Extract key endpoints
jq '{authorization_endpoint,token_endpoint,userinfo_endpoint,jwks_uri,registration_endpoint}'

# Fetch JWKS
curl $(jq -r .jwks_uri config.json) | jq .
```

### 3.2 Burp Suite OAuth Scanner
1. **Proxy** → Intercept all traffic during a complete OAuth login flow.
2. **Target** → right-click the AS host → **Scan** → **Audit** with the OAuth checks enabled (Burp Pro 2022+).
3. Review scanner findings: missing state, PKCE, open redirect candidates.
4. Supplement with manual tests for each finding class above.

### 3.3 Manual State Parameter Test (5 minutes)
```
1. Browser A: initiate OAuth login; intercept /authorize; note state value.
2. Browser B (different session): complete the callback with Browser A's code but Browser B's session.
3. If Browser B is authenticated as Browser A's user → state not validated.
```

### 3.4 Tool Stack

| Tool | Purpose | Source |
|------|---------|--------|
| Burp Suite Pro | Intercept, Scanner, JWT Editor | portswigger.net |
| jwt_tool | JWT attacks (alg:none, key confusion, claim tampering) | github.com/ticarpi/jwt_tool |
| OAuth-Scan (Burp extension) | Automated OAuth checks | BApp Store |
| PortSwigger Web Security Academy | OAuth labs | portswigger.net/web-security/oauth |
| oauth2c | CLI OAuth client for manual flow testing | github.com/cloudentity/oauth2c |

---

## 4. Finding Schema and CVSS Guide

### Finding Template
```
Title: [Attack class] in [Component] — [Impact in one line]
CVSSv3.1: [score] [vector]
CWE: [CWE-ID]
Affected endpoint: [URL]
Flow: [authorization code | implicit | hybrid | device | CIBA]
Steps to reproduce:
  1. ...
  2. ...
Impact: [account takeover | token theft | CSRF | privilege escalation]
Evidence: [screenshot / Burp request-response pair]
Remediation: [specific fix]
References: [RFC, OWASP ASVS control, PortSwigger advisory]
```

### CVSS Guidance

| Vulnerability | Typical CVSSv3.1 | Rationale |
|--------------|-----------------|-----------|
| CSRF → ATO via missing state | 8.8 High | Network, no priv, UI required, full account compromise |
| Open redirect → code theft | 7.4 High | Network, no priv, UI required, confidentiality high |
| Authorization code injection | 8.8 High | similar to CSRF ATO |
| PKCE absent on public client | 6.5 Medium | Requires code interception (MITM or log access) |
| JWT alg:none token forgery | 9.8 Critical | Network, no priv, no UI, full compromise |
| RS256→HS256 confusion | 9.1 Critical | Authenticated forgery of any user's token |
| Implicit flow token in Referer | 5.3 Medium | Passive leakage to third-party resources |
| Refresh token not rotated | 5.7 Medium | Requires prior token theft |
| Nonce not validated | 6.5 Medium | Requires captured ID token |

---

## 5. Defensive Checklist

- [ ] `state` generated per-request (≥128 bits random), verified on callback, single-use
- [ ] `redirect_uri` validated by exact string match only
- [ ] PKCE S256 enforced for all public clients; reject `plain`
- [ ] Authorization codes single-use, ≤10-minute lifetime
- [ ] Refresh tokens rotated on every use; old tokens invalidated immediately
- [ ] ID token validation: `iss`, `aud`, `exp`, `iat`, `nonce`, `alg`, signature
- [ ] JWT library configured with explicit algorithm allowlist; `none` rejected
- [ ] Implicit flow disabled; migrate to authorization code + PKCE
- [ ] `Referrer-Policy: no-referrer` on callback pages
- [ ] Token storage: `HttpOnly Secure SameSite=Strict` cookies, not `localStorage`

---

## 6. References

- RFC 6749 — OAuth 2.0 Authorization Framework
- RFC 7636 — PKCE
- RFC 9700 — OAuth 2.0 Security Best Current Practice (2025)
- OpenID Connect Core 1.0
- OWASP ASVS v5 — V3 Session Management, V8 Data Protection
- PortSwigger Web Security Academy — OAuth 2.0 vulnerabilities
- CVE-2022-22965 class — Spring OAuth token endpoint injection
