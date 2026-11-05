---
name: rce-db-approaches
description: Use when an authorized penetration testing engagement has landed SQL injection or database credentials and the operator needs the privilege-gated path from database access to OS command execution (RCE) or full data extraction. Covers MSSQL, MySQL/MariaDB, PostgreSQL, and Oracle plus the PHP application layer (PHAR deserialization, file-write-to-webshell, backup/restore injection, exposed Adminer). Every technique entry carries exactly six fields - required privilege gate, execution pattern, tooling with verified repo links, exact detection signal (event IDs, audit groups, log lines, canaries), and the hardening fix - so the offensive step and its defensive pairing ship together. Start from the decision tree in this file: match your privilege tier on the target DBMS and stop at the first reachable leaf; full technique detail lives in the references/ files.
---

# RCE-DB Approaches — privilege-gated paths from DB access to code execution

**Pairing rule (the skill's identity):** every offensive technique carries its required
privilege, its detection signal, and its hardening fix. Half of every entry is defensive.
An operator who can enable `xp_cmdshell` must be able to ship the audit rule and the
`sp_configure` reversal in the same deliverable.

## Engagement frame

Every technique here is published, documented tradecraft — MITRE ATT&CK **T1210**
(Exploitation of Remote Services), **T1505.003** (Web Shell), **T1059** (Command and
Scripting Interpreter), **T1005** (Data from Local System), **T1041** (Exfiltration Over
C2 Channel), **T1548** (Abuse Elevation Control Mechanism) — plus vendor hardening guides
and the public tool repositories cited in the reference files. The skill exists so an
authorized operator can BOTH execute a technique during an authorized engagement AND ship
the detection + fix in the same breath. Each entry states its REQUIRED PRIVILEGE
explicitly — low-privilege SQLi is never implied to be instant RCE.

All repo links in the references were verified live 2026-10-29; do not invent new ones.

## How to read an entry (the 6-field schema)

Every technique in the reference files carries exactly six fields:

| Field | Meaning |
|---|---|
| TECHNIQUE | Name + one-line mechanics |
| PRIVILEGE GATE | Exactly what the DB account needs — and what happens WITHOUT it |
| EXECUTION | The statement/command pattern, worked example |
| TOOLING | The corpus tool that automates it (repo link, one line) |
| DETECTION | The exact signal: event ID, log line, rule name, canary |
| FIX | The hardening: disable, restrict, least-privilege, config change |

## THE DECISION TREE — "I have [privilege] on [DBMS] → shortest path to RCE"

Start at your privilege level; stop at the first hit. Every leaf names the technique
detailed in the reference files (§ numbers map to the per-DBMS reference sections; the
PHP layer leaf labels map to its Part A–D structure).

### MSSQL → references/mssql-techniques.md

```
Do you have sysadmin (role membership or sa)?
├─ YES
│   ├─ xp_cmdshell enabled? ──────────────→ xp_cmdshell (§1) — instant
│   ├─ xp_cmdshell disabled (default 2005+)? → re-enable via sp_configure (§1)
│   │    (blocked only if the account cannot run sp_configure — sysadmin always can)
│   ├─ xp_cmdshell blocked by policy? ────→ OLE Automation sp_OACreate (§2)
│   ├─ CLR enabled? ──────────────────────→ CLR assembly, UNSAFE (§3)
│   ├─ SQL Agent running? ────────────────→ Agent CmdExec/PowerShell job (§4)
│   └─ All blocked? ──────────────────────→ read provider strings in linked servers (§7)
│                                            → impersonate across links → shell elsewhere
├─ NO, but an existing CmdExec/PowerShell Agent job is runnable by you
│   (SQLAgentOperatorRole, job ownership, or a mapped proxy) → §4
├─ NO, but IMPERSONATE on a privileged login
│   └─ EXECUTE AS LOGIN → sysadmin → restart above (§6)
├─ NO, but you can trigger UNC resolution (stacked queries, xp_dirtree/xp_subdirs)
│   └─ NetNTLM capture → relay/relay-to-HTTP → sysadmin elsewhere (§8)
├─ NO, but linked servers exist
│   ├─ Crawl links: EXECUTE AS at each hop (§7) → any hop with sysadmin → §1
│   └─ ADSI linked server readable? → domain creds (§5) → new foothold
└─ PLAIN LOGIN, no stacked queries, nothing above
    └─ No direct OS-exec path. Exfiltrate data; find creds in DB (agent
       passwords, linked-server logins, service creds in tables) → replay.
```

### MySQL / MariaDB → references/mysql-techniques.md

```
Do you have FILE privilege?
├─ NO
│   └─ No direct OS path on a stock server. Exfiltrate via UNION/error/OOB;
│      harvest creds from tables (wp_users, config tables) → app-level.
├─ YES — check @@secure_file_priv first (§3):
│   ├─ NULL (file ops disabled) ──────────→ dead end for file paths; creds-only
│   ├─ EMPTY (unrestricted — MariaDB default!) → best case:
│   │    ├─ plugin_dir writable? ─────────→ UDF RCE (§1, stage via §2)
│   │    ├─ webroot known & writable? ────→ INTO OUTFILE webshell (php-layer §2)
│   │    └─ Windows + UNC reachable? ─────→ SMB DLL load / NetNTLM capture (§4)
│   └─ SET TO A PATH — only that dir:
│        ├─ path == plugin_dir? ──────────→ UDF RCE (§1)
│        ├─ path == webroot? ─────────────→ INTO OUTFILE webshell (php-layer §2)
│        └─ else ─────────────────────────→ read/write only inside that dir (§6)
└─ MariaDB note: general_log file-write variant (§5) — gated by SUPER + log tables.
```

### PostgreSQL → references/postgres-techniques.md

```
Are you superuser?
├─ YES ──────────────────────────────────→ COPY ... TO PROGRAM (§1) — instant
├─ Member of pg_execute_server_program (PG14+ default-role)? → COPY PROGRAM (§1)
├─ Member of pg_write_server_files?
│   └─ write postgresql.conf (archive_command / ssl_passphrase_command)
│      → pg_reload_conf() → exec on next trigger (§4) — also persistence
├─ Member of pg_read_server_files? ──────→ read configs/creds (§3) → upgrade path
├─ REPLICATION privilege on PG < fixed-2026-08-13 builds? → CVE-2026-6471 dlopen (§5)
├─ dblink extension available? ──────────→ outbound TCP recon / NTLM capture (§2)
└─ PLAIN ROLE ───────────────────────────→ no server-side exec; OOB exfil via
                                            dblink/copy; large-object read
                                            only if granted (§3).
```

### Oracle → references/oracle-techniques.md

```
DBA-class account (or strong system privileges)?
├─ CREATE JOB + CREATE EXTERNAL JOB ─────→ DBMS_SCHEDULER EXECUTABLE job (§2)
├─ CREATE JAVA SOURCE + JVM installed ───→ DBMS_JAVA / Java stored proc exec (§1)
│   (21c+/23ai/26ai: in-DB Java (OJVM) NOT deprecated as of 2026-10-29 —
│    but MLE JavaScript is the strategic direction; check the estate, §6)
├─ DIRECTORY object with WRITE + UTL_FILE grant → file read/write (§3)
├─ Wide network ACL ─────────────────────→ UTL_HTTP/UTL_TCP OOB exfil (§4)
├─ No ACL grants? ───────────────────────→ ORA-24247 on every UTL_* call (§4)
└─ PLAIN ACCOUNT ────────────────────────→ no direct exec; harvest
                                            credentials/views; OOB only if ACL'd.
```

### PHP / application layer → references/php-layer-techniques.md (creds live IN the app)

```
What write primitive or endpoint do you have?
├─ File upload + fs-function touches phar:// + autoloaded gadget classes
│   └─ PHAR deserialization → RCE → read DB config (Part A) [PHP 8: explicit
│      Phar::getMetadata() only — see A1 trigger model]
├─ DB file-write primitive (any DBMS above) + writable webroot
│   └─ Webshell (Part B / B1) — extensions: .phtml/.user.ini/.htaccess (Part B gate matrix)
├─ Admin/backup import endpoint (CVE-2026-19949 class, All-in-One WP Migration ≤7.109)
│   └─ Malicious archive → restore → second-order SQLi → RCE (Part C)
├─ Trusted restored data consumed by sinks
│   └─ Second-order SQLi at use-site (Part C)
└─ Exposed adminer.php (no auth by default)
    ├─ 4.6.2-class version window → file read / NetNTLM (D1)
    ├─ 4.8.1-class version window → SSRF → metadata/creds (D2)
    ├─ Any version: form connects to ROGUE MySQL → cleartext cred capture (D3)
    └─ Captured creds → take them to the matching DBMS tree above
```

### FULL EXTRACTION — once you have any of the above (or even just DB read)

```
Native dump  : mysqldump / pg_dump / bcp or sqlcmd / expdp (Oracle Data Pump)
SQL-level    : SELECT ... INTO OUTFILE (MySQL), COPY TO (PG), bcp (MSSQL),
               spool (Oracle/SQL*Plus), orchestrate via information_schema loops
OOB exfil    : dblink (PG) · UTL_HTTP/UTL_TCP (Oracle) · DNS/HTTP via
               xp_cmdshell or shell (MITRE T1041) · Adminer dump-to-disk
Everything   : with OS exec → native client tools dump ALL databases/objects;
               without OS exec → per-DBMS extraction utilities + the db-takeover
               tool-layer report (SQLRecon "dump", enumdb, MSDAT).
```

## Section map (references/)

| File | Covers |
|---|---|
| `references/mssql-techniques.md` | xp_cmdshell, OLE Automation, CLR, Agent Jobs, ADSI, EXECUTE AS, linked servers, UNC capture |
| `references/mysql-techniques.md` | UDF chain, DUMPFILE hex-write, secure_file_priv gates, SMB DLL, log-write variants, LOAD_FILE harvest |
| `references/postgres-techniques.md` | COPY PROGRAM, dblink, large objects, conf-edit chain, CVE-2026-6471, untrusted-language extensions |
| `references/oracle-techniques.md` | DBMS_JAVA, DBMS_SCHEDULER, UTL_FILE, ACL exfil, password/config harvest, 2026 Java status |
| `references/php-layer-techniques.md` | PHAR deserialization + file-write→webshell + backup/restore injection + Adminer (merged) |

## Fold-in — companion layers (do not duplicate them here)

This skill is the **technique layer**. Two companion reports own adjacent layers:

- **`db-takeover`** (deep-research report, 2026-10-27) — the **tool layer**:
  SQLRecon, PySQLRecon, PowerUpSQL, mssql-spider, MSSQLand, SCCM takeover tooling,
  cloud-DB takeover. Tool entries here are one-line references, never re-documentation.
- **`sqli-surgeon`** (deep-research report, 2026-10-27) — the **injection-finding
  layer**: how you GET the SQLi / the DB credentials. This skill is what happens
  AFTER you have them.

If a task is "find the injection" → sqli-surgeon. If it is "which tool automates
this" → db-takeover. If it is "what can this privilege actually reach, and what
detects/fixes it" → this skill.
