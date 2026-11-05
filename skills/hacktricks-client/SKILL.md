---
name: hacktricks-client
description: HackTricks RAG API integration for authoritative penetration testing technique lookup. Returns relevant technique passages from HackTricks book, tool recommendations, and methodology guides via the ai.hacktricks.wiki RAG endpoint. Use when you need a specific exploitation technique, tool workflow, or CTF methodology that HackTricks covers.
triggers:
  - look up hacktricks
  - hacktricks technique
  - check hacktricks for
  - HackTricks reference
  - hacktricks query
  - search hacktricks
  - what does hacktricks say
  - hacktricks book
  - hacktricks methodology
  - hacktricks tool guide
---

# HACKTRICKS CLIENT — API Integration Skill

## Overview

HackTricks is the canonical penetration testing reference book (Carlos Polop, @carlospolop). The HackTricks RAG API provides programmatic access to the book's content via a semantic search endpoint. This skill covers authentication, query patterns, response parsing, quota management, and best-use patterns.

**API state (verified live 2026-10-30):**
- Weekly budget: **50 requests/week** (server-reported — overrides any claim of 100/week)
- Used as of 2026-10-30: 5 calls this week
- Remaining: **45/50** calls available
- Reset: Monday 00:00 Europe/Paris timezone
- Python client: `hacktricks_client.py` in `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/`

---

## API Endpoint and Authentication

```
Base URL:  https://ai.hacktricks.wiki/api/v1/
Endpoint:  POST /rag
Auth:      X-API-Key: <HACKTRICKS_API_KEY>
Content:   application/json
```

**Environment variable:** `HACKTRICKS_API_KEY`  
**Key format:** `htk_` prefix + 64 hex characters (67 chars total)  
**Key location (VPS):** `/srv/dr/secrets/api-keys.env` (chmod 600, chown rabi:rabi)  
**Key location (PC):** `C:\Users\rabim\OneDrive\Desktop\NEW RESULTS\03-KNOWLEDGE-BASE\01-API-CONFIG\api-keys.env`

---

## Query Format

### Request Body

```json
{
  "query": "how to exploit SQL injection with sqlmap",
  "top_k": 5,
  "min_score": 0.6
}
```

Parameters:
- `query` (required): Natural language query. Phrase as a question or technique description.
- `top_k` (optional, default 5): Number of passages to return. Keep at 3–5 to conserve quota.
- `min_score` (optional, default 0.5): Minimum relevance score. 0.6 filters noise well.

### Full curl Example

```bash
# Load key from env
source /srv/dr/secrets/api-keys.env

# Query HackTricks
curl -s -X POST "https://ai.hacktricks.wiki/api/v1/rag" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${HACKTRICKS_API_KEY}" \
  -d '{"query": "SSRF bypass techniques cloud metadata", "top_k": 3}' | python3 -m json.tool
```

### Response Format

```json
{
  "query": "SSRF bypass techniques cloud metadata",
  "results": [
    {
      "score": 0.89,
      "source": "pentesting-web/ssrf-server-side-request-forgery",
      "title": "SSRF - Server Side Request Forgery",
      "text": "To bypass IP filters in SSRF, you can use alternative IP representations: 0x7f000001 (hex), 0177.0.0.1 (octal), 127.0.1 (decimal shortform)..."
    }
  ],
  "quota": {
    "requests_used_this_week": 6,
    "requests_remaining": 44,
    "reset": "Monday 00:00 CET"
  }
}
```

---

## Python Client Usage

**Client file:** `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/hacktricks_client.py`

```python
from hacktricks_client import HackTricksClient

client = HackTricksClient()  # reads HACKTRICKS_API_KEY from env

# Simple query
results = client.query("Active Directory Kerberoasting attack")
for r in results:
    print(f"[{r['score']:.2f}] {r['title']}: {r['text'][:200]}")

# Query with custom parameters
results = client.query(
    "JWT alg none attack bypass",
    top_k=3,
    min_score=0.65
)

# Check remaining quota
quota = client.get_quota()
print(f"Remaining this week: {quota['remaining']}/{quota['limit']}")
```

---

## Rate Limit Management

**CRITICAL — 50 calls/week is tight. Conserve carefully.**

| Strategy | How |
|----------|-----|
| Check local KB first | Query `@knowledge-agent` / RAG index before calling HackTricks API |
| Batch queries | Combine related questions into one well-phrased query |
| Cache results | The client auto-caches responses; never call twice for the same query |
| Monitor quota | Call `client.get_quota()` before any session that will use multiple calls |
| Top-K discipline | Use `top_k=3` by default; only use 5 for broad technique surveys |

**Usage tracking:** `usage-tracker.json` records every call with timestamp and status code. Check it before large batch operations:

```bash
# On VPS
cat /srv/dr/projects/pentest-kb/usage-tracker.json | python3 -m json.tool | grep hacktricks -A 5
```

**Weekly reset:** Monday 00:00 Europe/Paris (CET/CEST). Plan bulk research for Monday–Tuesday when quota is full.

---

## Best-Use Patterns

### 1. Technique Lookup (most common)

```bash
# What you want: the exact technique steps for a known attack class
python3 -c "
from hacktricks_client import HackTricksClient
c = HackTricksClient()
for r in c.query('Kerberoasting step by step attack', top_k=3):
    print(f'=== {r[\"title\"]} ===')
    print(r['text'][:500])
    print()
"
```

### 2. Tool Discovery

```bash
# What you want: which tools HackTricks recommends for a task
python3 -c "
from hacktricks_client import HackTricksClient
c = HackTricksClient()
for r in c.query('tools for subdomain enumeration pentest', top_k=3):
    print(r['text'][:400])
"
```

### 3. CTF / Technique Identification

```bash
# What you want: identify a technique from symptoms
python3 -c "
from hacktricks_client import HackTricksClient
c = HackTricksClient()
for r in c.query('deserialization gadget chain Java ysoserial commons-collections', top_k=3):
    print(r['text'][:500])
"
```

### 4. CVE / Vulnerability Context

```bash
# What you want: HackTricks writeup for a known CVE class
python3 -c "
from hacktricks_client import HackTricksClient
c = HackTricksClient()
for r in c.query('ADCS ESC1 certificate template exploit Certipy', top_k=3):
    print(r['text'][:500])
"
```

---

## High-Value Query Examples

```python
# SQL injection DBMS-specific
"PostgreSQL SQL injection copy from program RCE"
"MSSQL xp_cmdshell SQL injection OS command execution"
"Oracle SQLi file read write techniques"

# WAF bypass
"WAF bypass SQL injection encoding techniques"
"Cloudflare WAF bypass SQL injection sqlmap tamper"

# XSS advanced
"DOM-based XSS innerHTML sources sinks"
"CSP bypass JSONP CDN Angular unsafe-eval"

# Authentication attacks
"JWT RS256 to HS256 key confusion attack"
"OAuth authorization code injection CSRF state parameter"

# Active Directory
"BloodHound attack paths Active Directory lateral movement"
"NTLM relay attack Responder ntlmrelayx"
"Pass the hash Windows lateral movement"

# Cloud
"AWS IAM privilege escalation assume role"
"GCP service account impersonation token theft"
"Azure managed identity IMDS credentials"

# Post-exploitation
"Linux privilege escalation SUID binaries"
"Windows privilege escalation token impersonation"
"LOLBAS living off the land Windows"
```

---

## Quota Conservation Decision Tree

```
Before calling HackTricks API:
    │
    ├─ Is the answer in the local RAG index?
    │   YES → use @knowledge-agent, save the call
    │   NO  → continue
    │
    ├─ Is it in the MASTER-REPORT.md (tools 5–10)?
    │   YES → read the file directly, save the call
    │   NO  → continue
    │
    ├─ Can I find it with a web search?
    │   YES → prefer web search for time-sensitive content
    │   NO  → continue
    │
    └─ Call HackTricks API (costs 1 quota unit)
        ├─ Cache the result in the local index
        └─ Update usage-tracker.json
```

---

## Health Check

```python
# Verify API is reachable and key is valid
from hacktricks_client import HackTricksClient
c = HackTricksClient()
quota = c.get_quota()
print(f"HackTricks API: OK | Remaining: {quota['remaining']}/{quota['limit']}")
```

Expected output: `HackTricks API: OK | Remaining: 44/50`

If 401 — key expired or env var not loaded. Check `/srv/dr/secrets/api-keys.env`.  
If 429 — weekly quota exhausted. Reset is Monday 00:00 CET.

---

## References

- HackTricks book: https://book.hacktricks.wiki/
- HackTricks AI RAG: https://ai.hacktricks.wiki/
- Client source: `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/hacktricks_client.py`
- Rate limits config: `03-KNOWLEDGE-BASE/01-API-CONFIG/rate-limits.json`
- Usage tracker: `03-KNOWLEDGE-BASE/01-API-CONFIG/usage-tracker.json`
- API config docs: `03-KNOWLEDGE-BASE/01-API-CONFIG/API-CONFIG.md`

*Authored: 2026-10-31 | Seat-40 knowledge-integrator | Quota verified live 2026-10-30*
