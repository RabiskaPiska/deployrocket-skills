---
name: github-tool-discovery
description: GitHub API tool discovery for security research — find offensive/defensive security tools, GitHub Security Advisories (GHSA), and trending repositories using anonymous (60 req/hr) or PAT-authenticated (5000 req/hr) access. Includes vault-first rule, stars+maintenance scoring, and search patterns for pentest tool discovery. Use when looking for new tools, checking if a CVE has a public PoC, or searching for tools by technique.
triggers:
  - find tools on GitHub
  - search GitHub for
  - discover new tools
  - GitHub security repos
  - GitHub tool search
  - find GitHub PoC
  - search GitHub advisory
  - GHSA lookup
  - GitHub security advisory
  - find exploit on GitHub
  - new pentest tools GitHub
---

# GITHUB TOOL DISCOVERY — API Skill

## Overview

This skill covers using the GitHub REST API v3 for security tool discovery, GitHub Security Advisory (GHSA) lookups, and trending repo identification. Two operation modes: anonymous (no key, 60 req/hr) and PAT-authenticated (5,000 req/hr core + 30 search/min).

**API state (verified live 2026-10-30):**
- GitHub PAT quota used: 15 calls today
- Remaining: **4,990/5,000** hourly requests
- Search remaining: ~30/30 per minute window
- Python client: `github_client.py` in `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/`

---

## Authentication Modes

### Anonymous (no PAT)
- Rate limit: **60 requests/hour** (shared per IP)
- Use for: one-off lookups, quick checks, when PAT is unavailable
- No env var needed

### PAT-Authenticated (recommended)
- Rate limit: **5,000 requests/hour** (core) + **30 search requests/minute**
- Environment variable: `GITHUB_PAT`
- PAT location (VPS): `/home/rabi/.env` — **never synced to git, never in any backup**
- PAT location (PC): `C:\Users\rabim\OneDrive\Desktop\NEW RESULTS\03-KNOWLEDGE-BASE\01-API-CONFIG\api-keys.env`
- Required scopes: `read:user` only — public advisories need no special scope
- Format: `github_pat_` prefix + alphanumeric string (~93 chars)

```bash
# Load PAT from env
source /srv/dr/secrets/api-keys.env
# or on VPS:
source /home/rabi/.env

# Verify authentication
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/rate_limit" | python3 -m json.tool | grep -A 5 '"core"'
```

---

## Vault-First Rule (MANDATORY)

**Always check the local vault before searching GitHub.**

The VPS has `/srv/dr/tools/` with tools already cloned. Searching GitHub and re-cloning a tool that's already installed wastes quota and time.

```bash
# Check vault first
ls /srv/dr/tools/ | grep -i <tool-name>

# If present → use it
# If absent → search GitHub → clone to /srv/dr/tools/<tool-name>
```

Tool index (when created): `/srv/dr/tools/tools-index.json`  
**Note:** As of 2026-10-31, `tools-index.json` was NOT created (Seat 51 blocked by VPS offline). Use `ls /srv/dr/tools/` directly.

---

## Search Patterns

### 1. Security Tool Search

```bash
# Find tools for a specific technique
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/repositories?q=sqlmap+tamper+scripts&sort=stars&order=desc&per_page=10" \
  | python3 -c "import sys,json; [print(f\"{r['full_name']} ({r['stargazers_count']}⭐) {r['description']}\") for r in json.load(sys.stdin)['items']]"

# WAF bypass tools
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/repositories?q=WAF+bypass+security&sort=stars&order=desc&per_page=10"

# Specific technique tools
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/repositories?q=SSRF+scanner+pentest&sort=updated&order=desc&per_page=10"
```

### 2. GHSA Advisory Search

```bash
# All WordPress/PHP advisories (high severity)
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/advisories?type=reviewed&ecosystem=composer&severity=high&per_page=30" \
  | python3 -c "
import sys, json
for a in json.load(sys.stdin):
    print(f\"{a.get('ghsa_id','?')} | {a.get('severity','?').upper()} | {a.get('summary','')[:80]}\")
"

# Search by CVE ID
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/advisories?cve_id=CVE-2026-19949" \
  | python3 -m json.tool

# NPM/JavaScript advisories
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/advisories?type=reviewed&ecosystem=npm&severity=critical&per_page=20"

# Python advisories
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/advisories?type=reviewed&ecosystem=pip&severity=high&per_page=20"
```

### 3. PoC / Exploit Search

```bash
# Find PoC for a specific CVE
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/repositories?q=CVE-2026-19949+poc+exploit&sort=stars&order=desc"

# Find PoCs across all CVEs (trending)
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/repositories?q=CVE+2026+PoC+exploit+pentest&sort=updated&order=desc&per_page=20"
```

### 4. Code Search (find technique implementations)

```bash
# Find actual tamper script implementations
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/code?q=tamper+WAF+bypass+sqlmap+language:python&per_page=10"

# Find wordlists
curl -s -H "Authorization: Bearer ${GITHUB_PAT}" \
  "https://api.github.com/search/repositories?q=pentest+wordlist+fuzzing&sort=stars&order=desc&per_page=10"
```

---

## Python Client Usage

**Client file:** `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/github_client.py`

```python
from github_client import GitHubClient

client = GitHubClient()  # reads GITHUB_PAT from env

# Search repositories
repos = client.search_repos("SSRF scanner pentest", sort="stars", per_page=10)
for r in repos:
    print(f"{r['full_name']} ({r['stars']}★) — last updated: {r['updated_at'][:10]}")
    print(f"  {r['description']}")
    print(f"  Archived: {r['archived']} | License: {r.get('license_spdx','none')}")

# Search advisories
advisories = client.search_advisories(ecosystem="composer", severity="high", per_page=20)
for a in advisories:
    print(f"{a['ghsa_id']} | {a['severity'].upper()} | {a['summary'][:80]}")

# Get rate limit status
limits = client.get_rate_limit()
print(f"Core: {limits['core']['remaining']}/{limits['core']['limit']}")
print(f"Search: {limits['search']['remaining']}/{limits['search']['limit']}")
```

---

## Stars + Maintenance Scoring

Before cloning a discovered tool, score it:

```python
def score_repo(repo: dict) -> float:
    """Score 0–1. ≥0.6 = worth cloning."""
    score = 0.0

    # Stars (35% weight)
    stars = repo.get('stargazers_count', 0)
    if stars >= 1000:   score += 0.35
    elif stars >= 500:  score += 0.25
    elif stars >= 100:  score += 0.15
    elif stars >= 10:   score += 0.05

    # Recent update (30% weight)
    from datetime import datetime, timezone
    updated = datetime.fromisoformat(repo['updated_at'].rstrip('Z')).replace(tzinfo=timezone.utc)
    days_old = (datetime.now(timezone.utc) - updated).days
    if days_old <= 90:   score += 0.30
    elif days_old <= 365: score += 0.20
    elif days_old <= 730: score += 0.10

    # License (20% weight) — prefer permissive
    license_id = (repo.get('license') or {}).get('spdx_id', '') or ''
    if license_id in ('MIT', 'Apache-2.0', 'BSD-3-Clause', 'BSD-2-Clause'):
        score += 0.20
    elif license_id in ('GPL-3.0', 'GPL-2.0', 'LGPL-2.1'):
        score += 0.10
    # No license: +0.0 — use at own risk, still clonable for research

    # Not archived (15% weight)
    if not repo.get('archived', False):
        score += 0.15

    return round(score, 2)
```

**Scoring thresholds:**
- ≥ 0.75 — high quality; clone immediately
- 0.60–0.74 — good; clone for technique coverage
- 0.40–0.59 — moderate; evaluate manually before cloning
- < 0.40 — low quality or unmaintained; skip unless unique technique

---

## Advisory Ecosystem Values

Valid `ecosystem` parameter values for the GHSA advisory endpoint:

```
actions       composer      erlang        go
maven         npm           nuget         other
pip           pub           rubygems      rust
swift
```

For WordPress/PHP work: use `composer` (Packagist ecosystem).  
For Node.js/frontend: use `npm`.  
For Python tools: use `pip`.

---

## Pagination Pattern

GitHub paginates at 30 items (default) or 100 items max per page. Always paginate for comprehensive searches:

```python
import requests, os, time

def paginate_search(query, max_pages=5):
    headers = {"Authorization": f"Bearer {os.getenv('GITHUB_PAT')}"}
    results = []
    for page in range(1, max_pages + 1):
        resp = requests.get(
            "https://api.github.com/search/repositories",
            headers=headers,
            params={"q": query, "sort": "stars", "order": "desc",
                    "per_page": 100, "page": page}
        )
        if resp.status_code == 429:
            time.sleep(60)  # search rate limit: wait 1 min
            continue
        items = resp.json().get("items", [])
        if not items:
            break
        results.extend(items)
        # Respect rate limit: 30 search req/min
        time.sleep(2)
    return results
```

---

## Rate Limit Error Handling

```python
import requests, time

def safe_github_get(url: str, headers: dict, retries: int = 3) -> dict:
    for attempt in range(retries):
        resp = requests.get(url, headers=headers, timeout=15)
        if resp.status_code == 200:
            return resp.json()
        elif resp.status_code == 403:
            # Check X-RateLimit-Remaining
            remaining = int(resp.headers.get('X-RateLimit-Remaining', 0))
            reset_at = int(resp.headers.get('X-RateLimit-Reset', time.time() + 60))
            if remaining == 0:
                wait = max(0, reset_at - time.time()) + 5
                print(f"[!] Rate limited. Waiting {wait:.0f}s...")
                time.sleep(wait)
        elif resp.status_code == 422:
            # Search query error — don't retry
            print(f"[!] Bad search query: {resp.json().get('message')}")
            return {}
        time.sleep(2 ** attempt)
    return {}
```

---

## High-Value Security Repos (Pre-Verified, Always Useful)

These are in the VPS vault (`/srv/dr/tools/`) or should be:

| Repo | Stars | Purpose |
|------|-------|---------|
| `swisskyrepo/PayloadsAllTheThings` | 65k+ | Canonical payload reference |
| `sqlmapproject/sqlmap` | 33k+ | SQL injection framework |
| `rapid7/metasploit-framework` | 39k+ | Exploitation framework |
| `ffuf/ffuf` | 13k+ | Web fuzzer |
| `projectdiscovery/nuclei` | 23k+ | Vulnerability scanner |
| `projectdiscovery/subfinder` | 12k+ | Subdomain enumeration |
| `projectdiscovery/httpx` | 8k+ | HTTP toolkit |
| `byt3bl33d3r/CrackMapExec` | 12k+ | AD/SMB toolkit |
| `BloodHoundAD/BloodHound` | 13k+ | AD attack-path visualization |
| `ly4k/Certipy` | 5k+ | ADCS exploitation |
| `dafthack/MSOLSpray` | 2k+ | Azure AD password spray |
| `wpscanteam/wpscan` | 9k+ | WordPress scanner |

---

## Weekly Advisory Scan Script

Run weekly to catch new high/critical advisories:

```bash
#!/bin/bash
# weekly-advisory-scan.sh
source /srv/dr/secrets/api-keys.env

echo "=== HIGH/CRITICAL GHSA ADVISORIES (last 7 days) ==="
for ecosystem in composer npm pip rubygems; do
    echo ""
    echo "--- $ecosystem ---"
    curl -s \
      -H "Authorization: Bearer ${GITHUB_PAT}" \
      "https://api.github.com/advisories?type=reviewed&ecosystem=${ecosystem}&severity=critical&per_page=10" \
      | python3 -c "
import sys, json
for a in json.load(sys.stdin):
    print(f\"  {a.get('ghsa_id','?')} | {a.get('summary','')[:70]}\")
    for cve in a.get('cve_id', []):
        print(f\"    CVE: {cve}\")
"
done
```

---

## References

- GitHub REST API docs: https://docs.github.com/en/rest
- GHSA advisories endpoint: https://docs.github.com/en/rest/security-advisories/global-advisories
- GitHub search syntax: https://docs.github.com/en/search-github/searching-on-github/searching-for-repositories
- Client source: `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/github_client.py`
- PAT on VPS: `/home/rabi/.env` (never synced)
- Secrets config: `03-KNOWLEDGE-BASE/01-API-CONFIG/API-CONFIG.md`
- Usage tracker: `03-KNOWLEDGE-BASE/01-API-CONFIG/usage-tracker.json`

*Authored: 2026-10-31 | Seat-40 knowledge-integrator | 4,990/5,000 hourly quota available*
