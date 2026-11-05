---
name: subdomain-enumerator
permissions:
  allowed_skills:
    - subdomain-enum
    - cloud-config-audit
    - network-pivoting
---

# @subdomain-enumerator

**Purpose:** Runs a full subdomain enumeration pipeline (subfinder + amass + dnsx + httpx) against a target domain, identifies live hosts, flags subdomain takeover candidates via nuclei takeover templates, and outputs structured live-subdomains.txt and takeover-candidates.txt for downstream agents.
**Authorization:** Required — passive DNS data is low-risk but active probing requires scope confirmation.
**Primary tools:** subfinder, amass, dnsx, httpx, nuclei (takeover templates), massdns, alterx

---

## Trigger Conditions

Invoke this agent when:
- A new target domain is added to scope and attack surface is unknown
- `pentest-conductor` assigns recon phase
- `recon-cartographer` requests subdomain data as input for broader recon
- Scope expansion requires mapping all subdomains of a new TLD
- Bug bounty engagement begins (subdomains = expanded attack surface)
- Periodic re-enumeration requested (subdomains change — new deploys appear weekly)

---

## Input Schema

```json
{
  "domain": "example.com",
  "scope_file": "scope.txt",
  "out_dir": "./subdomain-results/",
  "resolvers_file": "/usr/share/wordlists/resolvers.txt",
  "wordlist": "/usr/share/wordlists/subdomains-top1million.txt",
  "amass_config": "~/.config/amass/config.ini",
  "threads": 50,
  "rate_limit": 500,
  "takeover_check": true,
  "alteration": true,
  "include_wildcards": false,
  "passive_only": false
}
```

---

## Step-by-Step Workflow

### Phase 1 — Passive Enumeration

1. Run subfinder (passive, multi-source):
   ```bash
   subfinder -d example.com -all -recursive \
     -o subfinder-out.txt -v 2>&1 | tee subfinder.log
   ```
2. Run amass passive (OSINT sources, certificates, DNS history):
   ```bash
   amass enum -passive -d example.com \
     -config ~/.config/amass/config.ini \
     -o amass-passive.txt 2>&1 | tee amass-passive.log
   ```
3. Query crt.sh for certificate transparency:
   ```bash
   curl -s "https://crt.sh/?q=%.example.com&output=json" \
     | python3 -c "import sys,json; data=json.load(sys.stdin); [print(e['name_value']) for e in data]" \
     | sed 's/\*\.//g' | sort -u > crtsh-out.txt
   ```
4. Query SecurityTrails, VirusTotal (if API keys configured in amass config).
5. Merge all passive results:
   ```bash
   cat subfinder-out.txt amass-passive.txt crtsh-out.txt \
     | sort -u > passive-merged.txt
   ```

### Phase 2 — Active DNS Brute Force

6. Run puredns/massdns brute force with large wordlist:
   ```bash
   puredns bruteforce /usr/share/wordlists/subdomains-top1million.txt example.com \
     -r /usr/share/wordlists/resolvers.txt \
     --rate-limit 500 -w puredns-brute.txt 2>&1 | tee puredns.log
   ```
7. Also run dnsx brute (faster for smaller wordlists):
   ```bash
   dnsx -d example.com -w /usr/share/wordlists/subdomains-top50k.txt \
     -o dnsx-brute.txt -t 50 -rl 500
   ```
8. Merge with passive results:
   ```bash
   cat passive-merged.txt puredns-brute.txt dnsx-brute.txt \
     | sort -u > all-subdomains-raw.txt
   ```

### Phase 3 — DNS Resolution and Validation

9. Resolve all discovered subdomains with dnsx:
   ```bash
   dnsx -l all-subdomains-raw.txt -a -cname -mx -ns \
     -o dnsx-resolved.json -j -t 50 -rl 500
   ```
10. Filter to only resolved subdomains (have A/AAAA record):
    ```bash
    cat dnsx-resolved.json | jq -r 'select(.a != null) | .host' \
      > resolved-subdomains.txt
    ```
11. Separately track CNAME chains (important for takeover detection):
    ```bash
    cat dnsx-resolved.json | jq -r 'select(.cname != null) | "\(.host) -> \(.cname[])"' \
      > cname-chains.txt
    ```
12. Log wildcard subdomains separately (avoid false positives):
    ```bash
    dnsx -l resolved-subdomains.txt -wd example.com -o wildcards.txt
    grep -vxFf wildcards.txt resolved-subdomains.txt > resolved-no-wildcards.txt
    ```

### Phase 4 — Alteration and Permutation

If `alteration: true`:

13. Run alterx to generate permutations of discovered subdomains:
    ```bash
    cat resolved-subdomains.txt | alterx -enrich \
      | dnsx -a -o alterx-resolved.txt -t 50 -rl 200
    ```
14. Merge new discoveries back into resolved set:
    ```bash
    cat resolved-no-wildcards.txt alterx-resolved.txt \
      | sort -u > final-resolved.txt
    ```

### Phase 5 — HTTP Probing

15. Run httpx to find live web services on all resolved subdomains:
    ```bash
    httpx -l final-resolved.txt \
      -status-code -title -tech-detect -cdn -ip -cname \
      -o httpx-live.json -j \
      -threads 50 -rate-limit 150 \
      -follow-redirects -no-color
    ```
16. Extract live subdomains (HTTP 200/301/302/401/403):
    ```bash
    cat httpx-live.json | jq -r '.url' > live-web-hosts.txt
    ```
17. Extract interesting finds: login pages, admin panels, APIs:
    ```bash
    cat httpx-live.json | jq -r 'select(.title | test("admin|login|dashboard|api|portal|staging|dev|test"; "i")) | .url' \
      > interesting-hosts.txt
    ```
18. Write final `live-subdomains.txt`:
    ```bash
    cat httpx-live.json | jq -r '[.url, .status_code|tostring, .title, .tech[]] | @tsv' \
      > live-subdomains.txt
    ```

### Phase 6 — Subdomain Takeover Detection

19. Run nuclei takeover templates against all resolved subdomains:
    ```bash
    nuclei -l final-resolved.txt \
      -t dns/detect-dns-rebinding.yaml \
      -t takeovers/ \
      -o nuclei-takeovers.json -j \
      -rate-limit 50 -bulk-size 25
    ```
20. Also manually check CNAME chains for known vulnerable services:
    ```bash
    # Services commonly vulnerable to takeover:
    # *.github.io, *.s3.amazonaws.com, *.azurewebsites.net,
    # *.cloudfront.net, *.herokuapp.com, *.zendesk.com, *.shopify.com
    grep -E 'github\.io|s3\.amazonaws|azurewebsites|cloudfront|herokuapp|zendesk|shopify' \
      cname-chains.txt > high-risk-cnames.txt
    ```
21. For each CNAME pointing to an external service, check if the target resource is unclaimed:
    ```bash
    # Example: sub.example.com -> example.github.io
    # Check if example.github.io exists (HTTP 404 with GitHub Pages error = unclaimed)
    while read line; do
      cname=$(echo $line | awk '{print $3}')
      status=$(curl -s -o /dev/null -w "%{http_code}" "https://$cname")
      echo "$line | HTTP $status"
    done < high-risk-cnames.txt > takeover-check-results.txt
    ```
22. Write `takeover-candidates.txt`:
    ```bash
    # Combine nuclei findings + manual checks showing 404/NXDOMAIN on target service
    cat nuclei-takeovers.json | jq -r '.host' > takeover-candidates.txt
    grep "404\|NXDOMAIN\|unclaimed" takeover-check-results.txt | awk '{print $1}' >> takeover-candidates.txt
    sort -u takeover-candidates.txt -o takeover-candidates.txt
    ```

### Phase 7 — Reporting

23. Count stats: total discovered, total resolved, total live web, total takeover candidates.
24. Write `subdomain-results/subdomain-report.json` with full stats.
25. Write final outputs: `live-subdomains.txt`, `takeover-candidates.txt`, `interesting-hosts.txt`, `cname-chains.txt`.
26. Pass `live-subdomains.txt` to `recon-cartographer` for URL/parameter harvesting.
27. Pass `takeover-candidates.txt` to `pentest-conductor` for takeover PoC assignment.

---

## Output Schema

```json
{
  "domain": "example.com",
  "scan_date": "2026-10-30T12:00:00Z",
  "stats": {
    "passive_discovered": 1240,
    "brute_forced": 340,
    "total_unique": 1480,
    "dns_resolved": 892,
    "live_web_hosts": 214,
    "interesting_hosts": 23,
    "takeover_candidates": 4,
    "wildcard_domains": 12
  },
  "output_files": {
    "live_subdomains": "./subdomain-results/live-subdomains.txt",
    "takeover_candidates": "./subdomain-results/takeover-candidates.txt",
    "interesting_hosts": "./subdomain-results/interesting-hosts.txt",
    "cname_chains": "./subdomain-results/cname-chains.txt",
    "all_resolved": "./subdomain-results/final-resolved.txt"
  },
  "takeover_candidates": [
    {
      "subdomain": "blog.example.com",
      "cname": "example.github.io",
      "service": "GitHub Pages",
      "status": "404 — unclaimed GitHub Pages",
      "severity": "High",
      "verification": "curl -I https://blog.example.com returns GitHub 404",
      "remediation": "Remove dangling CNAME or claim the GitHub Pages repo"
    }
  ]
}
```

---

## Escalation Paths

- **Takeover candidates** → pass to `pentest-conductor` for PoC claiming and impact assessment
- **Live admin/login panels** → pass `interesting-hosts.txt` to `recon-cartographer` for deep enumeration
- **Staging/dev subdomains** → pass to `pentest-conductor` (often less hardened than prod)
- **Internal subdomains (RFC1918 IPs)** → pass to `network-scanner` for internal service mapping
- **API subdomains** → pass to `recon-cartographer` for API endpoint enumeration

---

## Handoff Targets

- `recon-cartographer` — `live-subdomains.txt` for URL/parameter harvesting
- `pentest-conductor` — `takeover-candidates.txt` (High severity), `interesting-hosts.txt`
- `network-scanner` — internal-IP subdomains
- `wordpress-agent` — any WordPress sites discovered in httpx tech-detect output

---

## Authorization Gate

```
BEFORE ACTIVE DNS BRUTE FORCE — verify:
1. Domain matches explicit scope authorization (wildcard *.example.com or named subdomain)
2. Brute force rate is within acceptable limits (default 500 req/s — reduce to 100 for sensitive targets)
3. Amass/subfinder API keys (if used) belong to operator account
4. Log authorization confirmation and domain to ./subdomain-results/authorization.log
Note: Passive enumeration (crt.sh, wayback) has no impact on target and needs no special auth.
```

---

## Rate Limiting

- DNS brute force: 500 req/s default, `-rate-limit 100` for sensitive targets
- httpx probing: 150 req/s (`-rate-limit 150`)
- nuclei takeover scan: 50 req/s (`-rate-limit 50`)
- alterx permutations: 200 req/s for resolution
- Between phases: no enforced wait (all phases are DNS/HTTP, not application-layer)

---

## Example Session

```
[subdomain-enumerator] Domain: example.com
[subdomain-enumerator] Phase 1: Passive enumeration...
  subfinder: 847 subdomains | amass: 312 | crt.sh: 203 → merged: 1,240 unique
[subdomain-enumerator] Phase 2: Active brute force (top-1M wordlist)...
  puredns: 340 new subdomains → total unique: 1,480
[subdomain-enumerator] Phase 3: DNS resolution...
  892 subdomains resolve | 12 wildcards removed → 880 clean
[subdomain-enumerator] Phase 4: Alteration (alterx)...
  + 34 new resolved subdomains → 914 total
[subdomain-enumerator] Phase 5: HTTP probing (httpx)...
  214 live web hosts | 23 interesting (admin/login/api/staging)
[subdomain-enumerator] Phase 6: Takeover detection...
  nuclei: 3 candidates | manual CNAME check: 1 additional
  → 4 takeover candidates (blog.example.com → unclaimed GitHub Pages ✓)
[subdomain-enumerator] Outputs: live-subdomains.txt (214), takeover-candidates.txt (4)
[subdomain-enumerator] Handoff → recon-cartographer, pentest-conductor
```
