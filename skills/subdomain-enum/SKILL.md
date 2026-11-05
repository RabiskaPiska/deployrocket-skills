---
id: subdomain-enum
name: subdomain-enum
description: >
  Subdomain enumeration specialist for authorized bug bounty and penetration testing.
  Covers subfinder/amass/assetfinder/dnsx/httpx pipeline, certificate transparency via
  crt.sh, DNS brute-force with wordlists, subdomain takeover detection with subjack and
  nuclei, cloud asset discovery, VHOST fuzzing with ffuf, and full automation scripts.
  Triggers on: subdomain enumeration, subdomain discovery, subfinder, amass, dnsx,
  subdomain takeover, VHOST fuzzing, certificate transparency, cloud subdomain.
triggers:
  - subdomain enumeration
  - subdomain discovery
  - subfinder
  - amass
  - dnsx
  - subdomain takeover
  - VHOST fuzzing
  - certificate transparency
  - cloud subdomain
  - assetfinder
  - httpx recon
  - attack surface mapping
---

# Subdomain Enumeration

## Authorization Gate

STOP. Confirm written authorization and defined scope before enumeration. Passive
techniques (crt.sh, CT logs) are lower risk; active DNS queries and VHOST fuzzing
generate noticeable traffic. Confirm what is in scope.

---

## 1. Quick Pipeline — Start Here

```bash
# Fastest complete pipeline for a new target
TARGET="example.com"
OUT="/tmp/recon/$TARGET"
mkdir -p $OUT

# Step 1: Passive gathering (all sources, no direct DNS)
subfinder -d $TARGET -all -recursive -o $OUT/subfinder.txt -silent &
assetfinder --subs-only $TARGET > $OUT/assetfinder.txt &
amass enum -passive -d $TARGET -o $OUT/amass_passive.txt 2>/dev/null &
wait

# Step 2: Certificate transparency
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | \
  python3 -c "import sys,json; [print(x['name_value']) for x in json.load(sys.stdin)]" | \
  tr ',' '\n' | sed 's/^\*\.//' | sort -u > $OUT/crt.txt

# Step 3: Merge and deduplicate
cat $OUT/subfinder.txt $OUT/assetfinder.txt $OUT/amass_passive.txt $OUT/crt.txt | \
  sed 's/^\*\.//' | sort -u > $OUT/all_subs.txt
wc -l $OUT/all_subs.txt

# Step 4: Resolve and filter live hosts
cat $OUT/all_subs.txt | dnsx -silent -a -aaaa -cname -mx -ns -resp -o $OUT/resolved.txt
cat $OUT/resolved.txt | awk '{print $1}' > $OUT/live_subs.txt

# Step 5: HTTP probe for live web services
cat $OUT/live_subs.txt | httpx -silent -title -status-code -tech-detect \
  -o $OUT/httpx_results.txt

echo "[+] Done. Live web services:"
wc -l $OUT/httpx_results.txt
```

---

## 2. Certificate Transparency (Passive, No Auth Required)

### 2.1 crt.sh Queries

```bash
TARGET="example.com"

# All certificates for domain (including wildcards)
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
names = set()
for entry in data:
    for name in entry['name_value'].split('\n'):
        name = name.strip().lstrip('*.')
        if name.endswith('.$TARGET'.replace('TARGET', '$TARGET')):
            names.add(name)
for n in sorted(names):
    print(n)
" 2>/dev/null | sort -u

# Deduplicated using jq
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | \
  jq -r '.[].name_value' | \
  sed 's/\*\.//g' | tr ',' '\n' | sort -u | \
  grep -v "^$" > crt_subs.txt

# Organization-based search (finds sibling domains)
ORG="Example Corp"
curl -s "https://crt.sh/?O=$ORG&output=json" | \
  jq -r '.[].name_value' | sort -u

# Google Certificate Transparency log via Censys (requires free API key)
# curl -s "https://search.censys.io/api/v2/certificates/search?q=parsed.names%3A$TARGET" \
#   -H "Authorization: Basic BASE64(API_ID:API_SECRET)"
```

### 2.2 Other CT Aggregators

```bash
# Facebook Certificate Transparency
curl -s "https://www.facebook.com/ct/api/certificate/?query=$TARGET&format=json&limit=1000" | \
  python3 -c "import sys,json; [print(x) for x in json.load(sys.stdin).get('data', {}).get('query', {}).get('results', [])]"

# Certspotter (Cert Spotter API — free tier)
curl -s "https://api.certspotter.com/v1/issuances?domain=$TARGET&include_subdomains=true&expand=dns_names" | \
  python3 -c "import sys,json; [print(n) for x in json.load(sys.stdin) for n in x.get('dns_names', [])]" | \
  sort -u
```

---

## 3. Subfinder — Fast Passive Enumeration

```bash
# Install
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# Basic scan
subfinder -d example.com -o subs.txt

# All sources (uses API keys from ~/.config/subfinder/provider-config.yaml)
subfinder -d example.com -all -o subs.txt

# Multiple domains
subfinder -dL domains.txt -all -o all_subs.txt

# Recursive enumeration
subfinder -d example.com -recursive -all -o recursive_subs.txt

# Silent mode for piping
subfinder -d example.com -silent | dnsx -silent | httpx -silent

# API key configuration (~/.config/subfinder/provider-config.yaml)
cat > ~/.config/subfinder/provider-config.yaml << 'EOF'
virustotal:
  - YOUR_VT_API_KEY
shodan:
  - YOUR_SHODAN_API_KEY
censys:
  - YOUR_CENSYS_API_ID:YOUR_CENSYS_API_SECRET
securitytrails:
  - YOUR_ST_API_KEY
EOF
```

---

## 4. Amass — Deep Enumeration

```bash
# Install
go install github.com/owasp-amass/amass/v4/...@latest

# Passive enumeration only (no DNS queries to target)
amass enum -passive -d example.com -o amass_passive.txt

# Active enumeration (DNS queries, brute force, permutations)
amass enum -active -d example.com -o amass_active.txt -config amass_config.ini

# With brute force wordlist
amass enum -brute -d example.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# Visualize attack surface
amass viz -d example.com -o amass_viz.dot
dot -Tpng amass_viz.dot -o attack_surface.png

# Amass config file for API keys
mkdir -p ~/.config/amass
cat > ~/.config/amass/config.ini << 'EOF'
[data_sources.VirusTotal]
apikey = YOUR_VT_API_KEY

[data_sources.Shodan]
apikey = YOUR_SHODAN_API_KEY

[data_sources.Censys]
apikey = YOUR_CENSYS_API_ID
secret = YOUR_CENSYS_API_SECRET
EOF
```

---

## 5. DNS Brute Force

### 5.1 dnsx — Fast DNS Toolkit

```bash
# Install
go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest

# Resolve list of subdomains
cat subs.txt | dnsx -silent -a -resp -o resolved.txt

# Brute force with wordlist
dnsx -d example.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -silent -a -o brute_results.txt

# Wildcard detection (critical — filter false positives)
dnsx -d example.com -wc -silent

# Resolve and get all record types
cat subs.txt | dnsx -silent -a -aaaa -cname -mx -ns -txt -resp -o full_dns.txt

# Filter only CNAME records (for takeover hunting)
cat subs.txt | dnsx -silent -cname -resp | grep CNAME > cname_records.txt
```

### 5.2 Puredns — Accurate Mass DNS Resolution

```bash
# Install
go install github.com/d3mondev/puredns/v2@latest

# Brute force (uses trusted resolvers to avoid false positives)
puredns bruteforce /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  example.com -o puredns_results.txt

# Resolve existing list accurately
puredns resolve subs.txt -o puredns_resolved.txt

# Use custom resolver list
puredns bruteforce wordlist.txt example.com \
  -r trusted_resolvers.txt \
  -o results.txt
```

### 5.3 Wordlist Recommendations

```bash
# Best wordlists for subdomain brute force (in order of effectiveness)
# 1. SecLists DNS
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt    # Fast, 5k
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt   # Good balance
/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt  # Thorough

# 2. DNS Words (permutation-aware)
# https://github.com/assetnote/wordlists
# commonspeak2 — generated from GitHub source code (30M unique words)

# 3. Generate permutations with gotator
go install github.com/Josue87/gotator@latest
gotator -sub subs.txt -perm permutations.txt -depth 1 -numbers 3 -silent | \
  dnsx -silent -o permutation_results.txt

# 4. DNS alteration with dnsgen  
pip install dnsgen
cat subs.txt | dnsgen - | dnsx -silent -o dnsgen_results.txt
```

---

## 6. Subdomain Takeover

### 6.1 Detection with subjack

```bash
# Install
go install github.com/haccer/subjack@latest

# Check all resolved subdomains for takeover potential
subjack -w live_subs.txt -t 100 -timeout 30 -ssl -c fingerprints.json -o takeover_candidates.txt

# Verbose output
subjack -w live_subs.txt -t 100 -timeout 30 -ssl -v -c fingerprints.json
```

### 6.2 Detection with Nuclei

```bash
# Nuclei subdomain takeover templates
nuclei -l live_subs.txt \
  -t http/takeovers/ \
  -t dns/detect-dangling-cname.yaml \
  -o nuclei_takeover.txt \
  -stats

# Specific cloud provider checks
nuclei -l live_subs.txt \
  -t http/takeovers/aws-s3-takeover.yaml \
  -t http/takeovers/azure-takeover.yaml \
  -t http/takeovers/github-pages-takeover.yaml \
  -t http/takeovers/heroku-takeover.yaml \
  -o cloud_takeovers.txt
```

### 6.3 Manual Takeover Checks

```bash
# Common dangling CNAME targets and their takeover services
declare -A takeover_map=(
  ["github.io"]="GitHub Pages — create repo at username.github.io"
  ["s3.amazonaws.com"]="AWS S3 — create bucket with same name"
  ["cloudfront.net"]="AWS CloudFront — requires more work"
  ["azurewebsites.net"]="Azure App Service — register subdomain"
  ["herokuapp.com"]="Heroku — create app with same name"
  ["fastly.net"]="Fastly — register CNAME in Fastly dashboard"
  ["ghost.io"]="Ghost — create blog with same name"
  ["shopify.com"]="Shopify — register store subdomain"
)

# Check CNAME chains for each subdomain
for sub in $(cat live_subs.txt); do
  cname=$(dig +short CNAME $sub 2>/dev/null | tail -1)
  if [ -n "$cname" ]; then
    echo "$sub → $cname"
    # Check if CNAME target resolves
    target_ip=$(dig +short A $cname 2>/dev/null)
    if [ -z "$target_ip" ]; then
      echo "  [POTENTIAL TAKEOVER] $sub → $cname (CNAME target doesn't resolve)"
    fi
  fi
done
```

---

## 7. Cloud Asset Discovery

### 7.1 S3 Bucket Enumeration

```bash
# Naming patterns for S3 buckets
TARGET="example"
declare -a patterns=(
  "$TARGET" "$TARGET-dev" "$TARGET-prod" "$TARGET-staging"
  "$TARGET-backup" "$TARGET-assets" "$TARGET-media" "$TARGET-static"
  "$TARGET-upload" "$TARGET-files" "$TARGET-logs" "$TARGET-data"
  "dev-$TARGET" "prod-$TARGET" "staging-$TARGET" "api-$TARGET"
)

for bucket in "${patterns[@]}"; do
  # AWS S3
  status=$(curl -so /dev/null -w "%{http_code}" "https://$bucket.s3.amazonaws.com/")
  echo "$status — $bucket.s3.amazonaws.com"
  
  # Regional buckets
  status=$(curl -so /dev/null -w "%{http_code}" "https://s3.amazonaws.com/$bucket")
  echo "$status — s3/$bucket"
done

# Automated: cloud_enum
pip install cloud-enum
cloud_enum -k TARGET -k example -k COMPANY_NAME \
  --disable-azure --disable-gcp

# S3Scanner
pip install s3scanner
s3scanner scan --bucket TARGET --bucket TARGET-prod --bucket TARGET-dev
```

### 7.2 Azure and GCP Discovery

```bash
# Azure blob storage
for name in $TARGET $TARGET-dev $TARGET-prod $TARGET-backup; do
  status=$(curl -so /dev/null -w "%{http_code}" "https://$name.blob.core.windows.net/")
  echo "$status — Azure: $name"
done

# GCP Cloud Storage
for name in $TARGET $TARGET-dev $TARGET-prod $TARGET-backup; do
  status=$(curl -so /dev/null -w "%{http_code}" "https://storage.googleapis.com/$name/")
  echo "$status — GCP: $name"
done

# GCP App Engine subdomains
for name in $TARGET $TARGET-dev $TARGET-api; do
  status=$(curl -so /dev/null -w "%{http_code}" "https://$name.appspot.com/")
  echo "$status — AppEngine: $name"
done
```

---

## 8. VHOST Fuzzing

### 8.1 ffuf VHOST Discovery

```bash
# Find virtual hosts on IP/CDN
# First get target IP
TARGET_IP=$(dig +short A example.com | head -1)

# VHOST fuzzing with ffuf
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://$TARGET_IP/ \
  -H "Host: FUZZ.example.com" \
  -fs 0 \
  -mc 200,201,301,302,403 \
  -o vhost_results.json

# HTTPS with SNI
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u https://$TARGET_IP/ \
  -H "Host: FUZZ.example.com" \
  -H "User-Agent: Mozilla/5.0" \
  -fs DEFAULT_SIZE \
  -mc 200,201,301,302,403 \
  -o vhost_https_results.json

# Filter by response size (first run without -fs to see default size)
# Then add: -fs DEFAULT_SIZE_FROM_FIRST_RUN

# gobuster VHOST mode
gobuster vhost -u https://example.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  --append-domain \
  -t 50
```

---

## 9. Httpx — Probing Live Hosts

```bash
# Install
go install github.com/projectdiscovery/httpx/cmd/httpx@latest

# Full probe with all useful flags
cat resolved_subs.txt | httpx \
  -silent \
  -title \
  -status-code \
  -tech-detect \
  -web-server \
  -content-length \
  -follow-redirects \
  -screenshot \
  -o httpx_full.json \
  -json

# Filter for interesting status codes
cat resolved_subs.txt | httpx -silent -status-code | grep -v "404\|000"

# Find admin panels
cat resolved_subs.txt | httpx -silent -path /admin -status-code -title

# Check for specific technologies
cat resolved_subs.txt | httpx -silent -tech-detect | grep -i "wordpress\|jenkins\|jira\|confluence"

# Find login pages
cat resolved_subs.txt | httpx -silent -title | grep -i "login\|sign in\|admin\|portal"
```

---

## 10. Full Automated Pipeline Script

```bash
#!/bin/bash
# Full subdomain enumeration pipeline
TARGET=$1
[ -z "$TARGET" ] && echo "Usage: $0 domain.com" && exit 1

OUT="./recon/$TARGET"
mkdir -p $OUT/{passive,active,resolved,live,takeover}

echo "[*] Starting recon for $TARGET"
echo "[*] Output: $OUT"

# Phase 1: Passive enumeration
echo "[1/7] Passive enumeration..."
subfinder -d $TARGET -all -recursive -silent -o $OUT/passive/subfinder.txt 2>/dev/null &
assetfinder --subs-only $TARGET > $OUT/passive/assetfinder.txt 2>/dev/null &
amass enum -passive -d $TARGET -o $OUT/passive/amass.txt 2>/dev/null &
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | \
  jq -r '.[].name_value' 2>/dev/null | tr ',' '\n' | sed 's/^\*\.//' | \
  sort -u > $OUT/passive/crt.txt
wait

# Phase 2: Merge
echo "[2/7] Merging results..."
cat $OUT/passive/*.txt | sed 's/^\*\.//' | \
  grep -E "^[a-zA-Z0-9._-]+\.$TARGET$" | \
  sort -u > $OUT/all_subs_raw.txt
echo "  [+] Unique subdomains: $(wc -l < $OUT/all_subs_raw.txt)"

# Phase 3: DNS brute force
echo "[3/7] DNS brute force..."
dnsx -d $TARGET \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -silent -a -o $OUT/active/brute.txt 2>/dev/null
cat $OUT/active/brute.txt | awk '{print $1}' >> $OUT/all_subs_raw.txt

# Phase 4: Resolve all
echo "[4/7] Resolving..."
sort -u $OUT/all_subs_raw.txt | \
  dnsx -silent -a -resp -o $OUT/resolved/all.txt 2>/dev/null
awk '{print $1}' $OUT/resolved/all.txt > $OUT/resolved/live.txt
echo "  [+] Resolved: $(wc -l < $OUT/resolved/live.txt)"

# Phase 5: HTTP probe
echo "[5/7] HTTP probing..."
httpx -l $OUT/resolved/live.txt -silent -title -status-code -tech-detect \
  -o $OUT/live/httpx.txt 2>/dev/null
echo "  [+] Live web: $(wc -l < $OUT/live/httpx.txt)"

# Phase 6: CNAME extraction for takeover
echo "[6/7] Checking for takeover candidates..."
dnsx -l $OUT/resolved/live.txt -cname -resp -silent 2>/dev/null | \
  grep CNAME > $OUT/takeover/cnames.txt
nuclei -l $OUT/resolved/live.txt -t http/takeovers/ -silent \
  -o $OUT/takeover/nuclei_takeover.txt 2>/dev/null

# Phase 7: Summary
echo "[7/7] Summary:"
echo "  Total subdomains found: $(sort -u $OUT/all_subs_raw.txt | wc -l)"
echo "  Resolved:               $(wc -l < $OUT/resolved/live.txt)"
echo "  Live web services:      $(wc -l < $OUT/live/httpx.txt)"
echo "  CNAME records:          $(wc -l < $OUT/takeover/cnames.txt)"
echo "  Takeover candidates:    $(wc -l < $OUT/takeover/nuclei_takeover.txt 2>/dev/null || echo 0)"
echo ""
echo "[+] Results saved to $OUT"
```

---

## 11. Shodan and Censys

```bash
# Shodan — find assets by organization/IP range
shodan search "org:\"Example Corp\"" --fields ip_str,port,hostnames,org | head -50
shodan search "ssl.cert.subject.cn:*.example.com" --fields ip_str,port,hostnames | head -50
shodan search "hostname:*.example.com" --fields ip_str,port,hostnames | head -100

# Censys
# https://search.censys.io/
# Query: parsed.names: "*.example.com"
# Returns all certificates referencing the domain

# SecurityTrails (API key required)
curl -s "https://api.securitytrails.com/v1/domain/$TARGET/subdomains" \
  -H "APIKEY: YOUR_API_KEY" | \
  jq -r '.subdomains[]' | sed "s/$/.${TARGET}/"
```

---

## Related Skills

- `recon-specialist` — full attack-surface recon including ports and services
- `cloud-attack` — exploiting discovered cloud assets
- `ssrf-specialist` — using discovered internal subdomains for SSRF
- `waf-bypass-techniques` — evading detection during active enumeration
