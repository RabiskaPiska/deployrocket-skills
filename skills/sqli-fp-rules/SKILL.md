---
id: sqli-fp-rules
name: SQLi False Positive Rules — Verification & Confirmed-TP Criteria
description: >
  False positive reduction and true positive confirmation for SQL injection
  findings. Applied after sqli-surgeon or sqli-shapeshifter output, or any
  time a SQLi finding needs verification before reporting. Covers 7 canonical
  FP patterns with per-pattern verification methodology and the confirmed-TP
  criteria required before a finding enters a report. Triggers on: "false
  positive", "verify SQLi", "confirm SQLi", "is this a real injection", after
  any sqli-surgeon or sqli-shapeshifter output.
triggers:
  - false positive
  - verify SQLi
  - confirm SQLi
  - is this real injection
  - sqli-surgeon output
  - sqli-shapeshifter output
  - SQLi verification
  - injection false positive
  - triage SQLi
version: "1.0"
author: fleet-seat-14
date: 2026-10-30
---

# SQLi False Positive Rules — Verification & Confirmed-TP Criteria

## Purpose

SQL injection scanners and even manual techniques produce false positives.
Reporting an FP destroys credibility with clients and bug bounty triagers.
This skill gates every SQLi finding through 7 canonical FP patterns before
the finding is allowed into a report. A finding that cannot pass the
confirmed-TP criteria for its technique MUST be marked "unverified — needs
further testing", never "confirmed".

---

## When to Invoke This Skill

- Immediately after any sqli-surgeon or sqli-shapeshifter session ends.
- Before writing any SQLi finding into a pentest report or bug bounty submission.
- When a finding "feels off" — unexpected response differences, inconsistent timing,
  payload reflecting in output without apparent DB interaction.
- When a client's dev team disputes a finding.

---

## The 7 FP Patterns

---

### FP-1: Error-Based FP on Version/Framework Strings

**What it looks like:**
The application returns a database error message containing what appears to be
a version string (e.g., `MySQL 5.7`, `SQL Server 2019`) when you inject a
quote or special character. Automated scanners flag this as error-based SQLi.

**Why it is often an FP:**
The application has generic error handling that echoes framework stack traces
regardless of whether the SQL query was actually affected. The error may be
triggered by input validation, middleware, or an ORM throwing an exception
before the query even reaches the database layer.

**Verification methodology:**
1. **Isolate the query path**: Send the same payload to 5 different parameters
   on the same endpoint. If ALL of them return the same error message, it is
   generic error handling, not injection.
2. **Check the error source**: Does the stack trace show the error originating
   from the DB driver, or from a validation layer? A `PDOException` at line 1
   of a validation class is not injection.
3. **Differential test**: Inject a semantically valid but structurally altered
   payload — e.g., `' OR '1'='1` vs `' OR '1'='2`. If the application response
   is IDENTICAL for both, the quote is not reaching the query logic.
4. **Try a non-error payload**: If you cannot elicit a behavioral difference
   (different rows returned, different response time, different content length)
   using a non-error payload, this is almost certainly FP.

**Confirmed-TP criteria for FP-1:**
Must demonstrate: (a) different application behavior for `'` vs `''` (escaped
quote) at the SAME parameter, AND (b) at least one of: different row count,
different column content, different timing, or OOB callback.

---

### FP-2: UNION FP on Column Count Mismatch

**What it looks like:**
A UNION-based injection attempt returns a generic error (`The used SELECT
statements have a different number of columns`) and the tool marks the
endpoint as "UNION injectable."

**Why it is often an FP:**
The column count error is returned by the DB engine, confirming the query
structure is reachable — but the scanner mistakes the error for a successful
UNION injection rather than a failed probe. The endpoint IS injectable, but
the specific UNION payload is not yet working. However, some tools also flag
column-count errors on endpoints where the error is from an ORM's own
type-checking, not from the DB.

**Verification methodology:**
1. **Confirm DB-origin of the error**: The error text must match known DB engine
   error strings (`MySQL: "different number of columns"`, `MSSQL: "All queries
   in an SQL statement containing a UNION operator must have an equal number"`,
   `PostgreSQL: "each UNION query must have the same number of columns"`).
   An ORM's own error will use different phrasing.
2. **Binary-search the column count**: Use `ORDER BY N` (increment N until error)
   to confirm the actual column count. If `ORDER BY` also throws a generic 500
   without following the DB's own error pattern for invalid ORDER BY, the query
   may not be reachable.
3. **Find the column count, then re-test UNION**: Once column count is confirmed,
   craft a matching UNION payload. If this returns data, it is a confirmed TP.
4. **Null-column test**: Use `UNION SELECT NULL,NULL,...` (matching count) with
   no payload content. If this returns a row without error, proceed to data
   extraction; the finding is confirmed injectable.

**Confirmed-TP criteria for FP-2:**
Must demonstrate: a successful `UNION SELECT NULL,...` (column-count matched)
that returns an extra row or modifies the response body in a reproducible way.
Column count error alone is NOT a confirmed TP.

---

### FP-3: Time-Based FP on High Server Load

**What it looks like:**
A `SLEEP(5)` or `WAITFOR DELAY '0:0:5'` injection causes the response to take
5–10 seconds, which the tool records as a confirmed time-based blind SQLi.

**Why it is often an FP:**
Under high server load, CDN throttling, slow network links, or during
peak traffic windows, ANY request can take 5+ seconds. The delay observed
may have nothing to do with the injected payload.

**Verification methodology:**
1. **Baseline first**: Send the exact same request WITHOUT the sleep payload
   five times and record response times. If baseline is already 2–4 seconds,
   a 5-second sleep is indistinguishable from noise.
2. **Delta test**: Compare `SLEEP(0)` vs `SLEEP(5)` vs `SLEEP(10)`. A genuine
   time-based injection will produce response times that scale linearly
   (0s → ~0s, 5s → ~5s, 10s → ~10s, within ±1s). Random server lag will
   NOT scale linearly.
3. **Repeat minimum 3 times**: Run the `SLEEP(5)` payload 3 times. If the delay
   is genuine, all 3 will be within ±1s of 5 seconds above baseline. If 1 of 3
   is slow, it is likely server load.
4. **Try a conditional sleep**: `IF(1=1, SLEEP(5), SLEEP(0))` vs
   `IF(1=2, SLEEP(5), SLEEP(0))`. A genuine injection will sleep on the true
   branch only. If both sleep, the parameter is not controlling the condition.
5. **Alternative: use `SLEEP(0.5)` repeating**: Many FP environments cannot
   consistently produce sub-second artificial delays. 10x `SLEEP(0.5)` summing
   to 5s is a stronger signal than one `SLEEP(5)`.

**Confirmed-TP criteria for FP-3:**
Must demonstrate: (a) linear scaling across at least 3 sleep values (1s, 5s, 10s),
AND (b) a successful conditional sleep where true-branch sleeps and false-branch
does not, run at least twice.

---

### FP-4: Boolean-Based FP on Application-Level Caching

**What it looks like:**
Boolean blind testing with `AND 1=1` vs `AND 1=2` shows different response
content — the tool concludes the parameter is boolean-injectable.

**Why it is often an FP:**
Application-level caches (Varnish, Redis, CDN edge caches) may serve cached
responses for the first request and a fresh (different) response for the second.
The difference in response bodies has nothing to do with the SQL condition.

**Verification methodology:**
1. **Bust the cache first**: Add a random cache-busting query parameter
   (e.g., `?cb=<random>`) to BOTH the true and false requests. If the difference
   disappears, it was a caching artifact.
2. **Reverse the condition**: Test `AND 2=2` (true) and `AND 2=3` (false).
   If the same pages respond differently, the boolean logic is reaching the DB.
3. **Try a non-injectable parameter**: Apply the same `AND 1=1` / `AND 1=2`
   to a parameter that logically cannot be injectable (e.g., a static CSS URL
   parameter). If you also get a response difference, it is caching.
4. **Cookie / session variance**: Check whether the response difference correlates
   with session state rather than the injected condition. Log out and test again.

**Confirmed-TP criteria for FP-4:**
Must demonstrate: consistent response difference for `AND 1=1` vs `AND 1=2`
across at least 5 requests each with cache-busting applied, AND the reversal
test (`AND 2=2` vs `AND 2=3`) produces the same direction of difference.

---

### FP-5: Out-of-Band (OOB) FP on DNS Noise

**What it looks like:**
An OOB payload (`LOAD_FILE`, `UTL_HTTP`, `xp_cmdshell` DNS callback) triggers
a DNS lookup to your Burp Collaborator / interactsh server, and the tool
reports confirmed OOB SQLi.

**Why it is often an FP:**
The DNS lookup may come from a security scanner, WAF, URL-preview service,
Slack bot, or the application's own link-preview feature scanning user-submitted
URLs — not from the database engine. This is particularly common when the payload
URL appears in a context where the application itself previews links (chat apps,
CMS editors, email systems).

**Verification methodology:**
1. **Source IP check**: Record the IP of the DNS lookup. Compare it to the
   application server's IP range (use the target's WHOIS/rDNS). If the source
   is a Cloudflare IP, Google bot range, or unrelated CDN — it is a scanner FP.
2. **Payload uniqueness test**: Use a collaborator subdomain that appears NOWHERE
   else (no copy-paste, no browser history). If you still get a lookup
   immediately on page load (before the payload is submitted), something else
   is scanning your collaborator domain.
3. **DB-specific payload test**: Use a DB-engine-specific OOB technique:
   MySQL: `LOAD_FILE(CONCAT('\\\\',<collab>,'\\share'))`
   MSSQL: `EXEC master..xp_dirtree '//<collab>/share'`
   PostgreSQL: `COPY (SELECT '') TO PROGRAM 'nslookup <collab>'`
   Oracle: `SELECT UTL_HTTP.REQUEST('http://<collab>') FROM DUAL`
   If the lookup arrives and you can confirm the DB type matches the technique
   used (e.g., MSSQL xp_dirtree on a confirmed MSSQL target), confidence is high.
4. **Timing correlation**: The DNS callback should arrive within the HTTP response
   time window. A callback arriving 30+ seconds after the request may be an async
   scanner, not synchronous DB execution.

**Confirmed-TP criteria for FP-5:**
Must demonstrate: (a) DNS/HTTP callback with source IP in the application server's
IP range, AND (b) DB-engine-specific payload used that matches the confirmed DBMS,
AND (c) callback timing correlates with the HTTP request (within ±3 seconds).

---

### FP-6: Reflected Input FP (Input Echoed Without DB Interaction)

**What it looks like:**
A payload like `' UNION SELECT 1,2,3 --` appears in the application's response
body. The tool flags this as UNION injection with visible output.

**Why it is often an FP:**
Many applications reflect user input back in error messages, search results
(showing "You searched for: [input]"), or debug output — without the input ever
reaching a SQL query. The reflected payload text is the application echoing your
input, not the DB returning UNION results.

**Verification methodology:**
1. **Content test**: Does the response contain ONLY your injected text, or does
   it also contain data from the database? If you see `1` and `2` and `3` in the
   response but these are your own payload values, there is no DB data leaking.
2. **Substitute DB data**: Change the payload to `UNION SELECT user(), version(),
   database()`. If the response shows MySQL function OUTPUT (e.g., `root@localhost`,
   `8.0.31`, `webapp`) instead of the literal strings `user()`, `version()`,
   `database()` — it is a confirmed TP.
3. **Null columns**: Use `UNION SELECT NULL,NULL,NULL`. If the application echoes
   `NULL,NULL,NULL` as text, it is reflecting your input. If it echoes empty
   values or omits the column, the UNION may be working but the column is NULL.
4. **Check the Content-Type**: If the reflected content is inside a JavaScript
   context or JSON response, the application may be building a JS object from
   your input — potentially an XSS surface, but NOT SQLi.

**Confirmed-TP criteria for FP-6:**
Must demonstrate: DB function or DB data appearing in the response — user(),
version(), database() output, actual table data, or a DB-specific string that
cannot come from the application itself.

---

### FP-7: Second-Order FP on Stored-Then-Processed Inputs

**What it looks like:**
A payload is stored in the application (e.g., in a username or profile bio),
and a separate page that displays or processes that stored value shows what
appears to be SQLi behavior.

**Why it is often an FP:**
The application may be HTML-encoding or escaping the stored value at retrieval
time (correct behavior), making it appear the payload is being injected when in
fact the displayed value is harmless encoded text. Some scanners mistake
`&lt;script&gt;` in the DOM as a stored XSS/SQLi candidate when it is just
properly escaped output.

**Verification methodology:**
1. **Inspect the raw response (not rendered HTML)**: In Burp Repeater, look at
   the raw HTTP response body. Is your payload present as `'` (literal apostrophe)
   or as `&#39;` / `&apos;` (HTML entity)? Encoded = the application is escaping,
   likely not injectable.
2. **Check the DB storage**: If you have DB read access (white-box), query the
   table directly. Is the stored value the raw payload or an already-escaped
   version? If the application escapes before storage, second-order injection
   is impossible unless the re-use point un-escapes.
3. **Find the re-use point**: Second-order SQLi requires the stored value to be
   used in a SQL query at a DIFFERENT point in the application (e.g., stored as
   username, used unsanitized in a query on the password reset page). If you
   cannot identify the re-use point, you do not have a confirmed finding.
4. **Trigger the re-use point**: With the payload stored, trigger the functionality
   that re-uses it. If the application behaves differently (extra data, errors,
   timing) in a way that correlates with your payload, proceed to full
   confirmation using FP-3 or FP-4 methodology on the re-use endpoint.

**Confirmed-TP criteria for FP-7:**
Must demonstrate: (a) raw payload (not encoded) stored in the DB or confirmed
retrievable un-escaped, AND (b) the re-use endpoint identified and triggered,
AND (c) observable DB behavioral difference at the re-use point verified using
at least one of the Boolean (FP-4) or Time-based (FP-3) confirmation methods.

---

## Confirmed-TP Criteria Summary Table

| Technique | Minimum Evidence Required to Mark as Confirmed TP |
|-----------|--------------------------------------------------|
| Error-based | Different behavior for `'` vs `''` at same param + at least one behavioral difference |
| UNION-based | Successful NULL-matched UNION returning modified response |
| Boolean blind | 5+ consistent differential responses with cache-busting + reversal test |
| Time-based blind | Linear scaling across 3 sleep values + conditional sleep (true vs false branch) |
| OOB / DNS | Source IP in app server range + DB-specific technique + timing correlation |
| Reflected input | DB function output (not literal strings) visible in response |
| Second-order | Raw payload stored + re-use point triggered + behavioral difference confirmed |

**Rule**: If a finding passes ALL criteria for its technique, it is a Confirmed TP.
If it passes SOME but not all, it is "Probable TP — needs additional evidence".
If it fails any criteria, it is "Unverified — do not report without further testing".

---

## Quick Triage Checklist (Run on Every SQLi Finding)

```
[ ] 1. Identified which of the 7 FP patterns this finding could match
[ ] 2. Ran the verification steps for that FP pattern
[ ] 3. Applied the confirmed-TP criteria — passed all required checks
[ ] 4. Tested the finding on at least 2 separate requests (reproducibility)
[ ] 5. Documented: exact payload, exact request, exact response diff
[ ] 6. Noted the DBMS, injection point type (GET/POST/header/cookie), and
        context (WHERE clause, INSERT, UPDATE, ORDER BY, LIMIT)
[ ] 7. Cross-checked against sqli-fp-rules — all 7 FP patterns considered
```

Only after all 7 boxes are checked does the finding move to the report.
