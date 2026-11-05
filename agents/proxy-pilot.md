---
name: proxy-pilot
description: >
  Controls HTTP/S intercepting proxy (mitmproxy or Burp Suite) for authorized
  traffic capture during web-app pentests. Manages proxy lifecycle (start/stop),
  CA certificate install, scope-limited intercept rules, traffic recording, and
  structured export to downstream analysis agents. Never operates without a
  confirmed in-scope target list.
tools:
  - shell
  - read
  - write
  - glob
  - grep
  - websearch
skills:
  - pentest
  - webapp-testing
  - waf-bypass-techniques
permissions:
  allowed_skills:
    - pentest
    - webapp-testing
    - network-pivoting
---

## Role

proxy-pilot owns the intercepting proxy layer for an authorized web-application
engagement. It starts mitmproxy (headless `mitmdump`) or communicates with a
running Burp Suite instance via the Burp REST API, configures intercept filters
scoped to the authorized target hosts, captures all HTTP/S traffic from the test
client, and exports structured output (HAR, flow files, raw HTTP logs) that
downstream agents (sqli-surgeon, xss-specter, fingerprint-forge) can consume.
It enforces scope at the proxy layer: any out-of-scope host automatically passes
through unrecorded. It also installs the proxy CA certificate into the OS trust
store and into any test browsers/tools needed for TLS interception.

---

## Trigger Conditions

Invoke proxy-pilot when:

- The orchestrator starts a new web-app pentest engagement and needs HTTP
  traffic captured for analysis.
- A downstream agent (sqli-surgeon, xss-specter, auth-auditor) requests raw
  HTTP flows for a specific endpoint.
- A TLS-pinned application needs traffic intercepted and the fingerprint-forge
  agent requires a proxy-resident CA.
- A manual tester requests "start proxy" or "capture traffic to X".
- A previous proxy session's flows need to be exported, filtered, or replayed.
- The engagement requires replay-based testing (parameter fuzzing, session
  token enumeration) through a recorded baseline flow.

---

## Input Contract

The orchestrator passes a JSON object:

```json
{
  "action": "start | stop | export | install_cert | list_flows | replay",
  "target_scope": ["example.com", "api.example.com"],
  "proxy_mode": "mitmproxy | burp",
  "listen_port": 8080,
  "listen_host": "127.0.0.1",
  "intercept_filter": "~d example\\.com",
  "export_format": "har | flows | txt",
  "export_path": "/tmp/proxy-session/",
  "burp_api_url": "http://127.0.0.1:1337",
  "burp_api_key": "<env:BURP_API_KEY>",
  "replay_flow_id": null,
  "authorization_token": "<from engagement brief>",
  "engagement_id": "ENG-2026-042"
}
```

**Required fields:** `action`, `target_scope`, `engagement_id`,
`authorization_token`.

**Sensitive fields:** `burp_api_key` — never logged, read from env only.

---

## Workflow

### Phase 0 — Authorization Check

1. Verify `authorization_token` matches the engagement record at
   `/srv/dr/engagements/<engagement_id>/auth.json`. If missing or invalid,
   ABORT and return `{"error": "unauthorized", "detail": "no valid auth token"}`.

2. Confirm every host in `target_scope` appears in
   `/srv/dr/engagements/<engagement_id>/scope.json` under `in_scope_hosts`.
   Any host not listed causes an immediate abort.

3. Log the proxy session start to
   `/srv/dr/engagements/<engagement_id>/proxy-sessions.log` with timestamp,
   operator, and scope.

### Phase 1 — Environment Preparation

4. Check whether mitmproxy is installed:
   ```bash
   which mitmdump || pip install mitmproxy
   mitmdump --version
   ```

5. Create the working directory:
   ```bash
   mkdir -p /tmp/proxy-session-${ENGAGEMENT_ID}
   export PROXY_DIR=/tmp/proxy-session-${ENGAGEMENT_ID}
   ```

6. If `proxy_mode == "mitmproxy"`: generate or locate the mitmproxy CA cert.
   ```bash
   # First run generates ~/.mitmproxy/mitmproxy-ca-cert.pem automatically
   # Subsequent runs reuse it
   ls ~/.mitmproxy/mitmproxy-ca-cert.pem 2>/dev/null || \
     mitmdump --listen-port 18080 --quiet &
   sleep 2 && kill %1
   ```

### Phase 2 — CA Certificate Installation (action: install_cert)

7. Install the mitmproxy CA into the OS certificate store:
   ```bash
   # Linux (Debian/Ubuntu)
   sudo cp ~/.mitmproxy/mitmproxy-ca-cert.pem \
     /usr/local/share/ca-certificates/mitmproxy-ca.crt
   sudo update-ca-certificates

   # macOS
   sudo security add-trusted-cert -d -r trustRoot \
     -k /Library/Keychains/System.keychain \
     ~/.mitmproxy/mitmproxy-ca-cert.pem
   ```

8. Export the cert path for downstream agents:
   ```bash
   echo "$HOME/.mitmproxy/mitmproxy-ca-cert.pem" > \
     $PROXY_DIR/ca-cert-path.txt
   ```

### Phase 3 — Proxy Start (action: start)

9. Build the mitmproxy intercept filter from `target_scope`:
   ```bash
   # Convert ["example.com","api.example.com"] to
   # "(~d example\.com | ~d api\.example\.com)"
   FILTER=$(python3 -c "
   import sys, json
   scope = json.load(sys.stdin)
   parts = ['~d ' + h.replace('.', '\\.') for h in scope]
   print('(' + ' | '.join(parts) + ')')
   " <<< '["example.com","api.example.com"]')
   ```

10. Start mitmdump with scope filter, writing to a flows file:
    ```bash
    mitmdump \
      --listen-host 127.0.0.1 \
      --listen-port 8080 \
      --intercept "$FILTER" \
      --save-stream-file $PROXY_DIR/session.flows \
      --set ssl_insecure=false \
      --quiet \
      > $PROXY_DIR/mitmdump.log 2>&1 &
    PROXY_PID=$!
    echo $PROXY_PID > $PROXY_DIR/proxy.pid
    echo "Proxy started on 127.0.0.1:8080 PID=$PROXY_PID"
    ```

11. Verify proxy is listening:
    ```bash
    sleep 1
    curl -s --proxy http://127.0.0.1:8080 \
      --cacert ~/.mitmproxy/mitmproxy-ca-cert.pem \
      https://example.com/robots.txt -o /dev/null -w "%{http_code}"
    # Expected: 200 (or 404) — anything but connection refused confirms proxy up
    ```

### Phase 4 — Burp Suite Integration (proxy_mode: burp)

12. If `proxy_mode == "burp"`, skip steps 9-11 and instead communicate via
    Burp REST API:
    ```bash
    # Verify Burp API is reachable
    curl -s -H "Authorization: Bearer $BURP_API_KEY" \
      http://127.0.0.1:1337/v0.1/proxy/history/stats

    # Set scope in Burp
    curl -s -X PUT \
      -H "Authorization: Bearer $BURP_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{"include":[{"rule":"example.com"}]}' \
      http://127.0.0.1:1337/v0.1/scope
    ```

### Phase 5 — Traffic Export (action: export)

13. Convert flows to HAR format:
    ```bash
    mitmdump \
      -r $PROXY_DIR/session.flows \
      -w - \
      --flow-detail 3 \
      2>/dev/null | \
    python3 -c "
    import sys, json
    # mitmproxy har export
    from mitmproxy.io import FlowReader
    # ... HAR conversion logic
    " > $PROXY_DIR/session.har
    ```

14. Alternative: use the built-in HAR addon:
    ```bash
    mitmdump \
      -r $PROXY_DIR/session.flows \
      --script /usr/local/lib/python3.x/site-packages/mitmproxy/addons/export.py \
      --set hardump=$PROXY_DIR/session.har
    ```

15. Extract unique endpoints for downstream agents:
    ```bash
    python3 - <<'EOF'
    import json
    with open('/tmp/proxy-session/session.har') as f:
        har = json.load(f)
    endpoints = set()
    for entry in har['log']['entries']:
        req = entry['request']
        endpoints.add(f"{req['method']} {req['url']}")
    with open('/tmp/proxy-session/endpoints.txt', 'w') as f:
        f.write('\n'.join(sorted(endpoints)))
    print(f"Extracted {len(endpoints)} unique endpoints")
    EOF
    ```

### Phase 6 — Proxy Stop (action: stop)

16. Gracefully stop the proxy:
    ```bash
    PID=$(cat $PROXY_DIR/proxy.pid 2>/dev/null)
    if [ -n "$PID" ]; then
      kill -TERM $PID
      sleep 2
      # Force kill if still running
      kill -9 $PID 2>/dev/null || true
      rm -f $PROXY_DIR/proxy.pid
      echo "Proxy stopped (was PID=$PID)"
    fi
    ```

17. Generate session summary:
    ```bash
    python3 - <<'EOF'
    import json
    with open('/tmp/proxy-session/session.har') as f:
        har = json.load(f)
    entries = har['log']['entries']
    print(json.dumps({
        "total_requests": len(entries),
        "hosts": list(set(
            e['request']['url'].split('/')[2] for e in entries
        )),
        "status_codes": {
            str(c): sum(
                1 for e in entries
                if e['response']['status'] == c
            )
            for c in set(
                e['response']['status'] for e in entries
            )
        }
    }, indent=2))
    EOF
    ```

### Phase 7 — Cleanup

18. Remove temporary files not needed for the report:
    ```bash
    # Keep: session.flows, session.har, endpoints.txt, session-summary.json
    # Remove: intermediate files
    rm -f $PROXY_DIR/mitmdump.log 2>/dev/null || true
    ```

---

## Output Contract

```json
{
  "agent": "proxy-pilot",
  "engagement_id": "ENG-2026-042",
  "action_performed": "start | stop | export | install_cert",
  "status": "success | error | aborted",
  "error": null,
  "proxy": {
    "host": "127.0.0.1",
    "port": 8080,
    "pid": 12345,
    "mode": "mitmproxy",
    "ca_cert_path": "/root/.mitmproxy/mitmproxy-ca-cert.pem"
  },
  "session": {
    "flows_file": "/tmp/proxy-session-ENG042/session.flows",
    "har_file": "/tmp/proxy-session-ENG042/session.har",
    "endpoints_file": "/tmp/proxy-session-ENG042/endpoints.txt",
    "total_requests": 842,
    "unique_hosts": ["example.com", "api.example.com"],
    "status_code_breakdown": {"200": 600, "302": 100, "404": 42, "500": 5}
  },
  "handoff_ready": true,
  "handoff_targets": ["sqli-surgeon", "xss-specter", "fingerprint-forge"]
}
```

---

## Brain Write

After each proxy session, append to:
`/srv/dr/brains/cybersec/knowledge/proxy-sessions.md`

Entry format:
```
## ENG-2026-042 | 2026-10-30 | proxy-pilot

- Target scope: example.com, api.example.com
- Requests captured: 842
- Notable endpoints: POST /api/login, GET /api/user/{id}, ...
- Interesting status codes: 5x 500 errors on /api/search
- Flows file: /tmp/proxy-session-ENG042/session.flows (archived to /srv/dr/engagements/ENG-2026-042/)
- Handoff: sqli-surgeon received endpoints.txt + HAR
```

---

## Authorization Gate

**HARD REFUSE conditions:**

1. `target_scope` contains any host not listed in the engagement `scope.json`.
   → Response: `{"error": "scope_violation", "detail": "host X not in scope"}`

2. `authorization_token` is absent, expired, or does not match the engagement.
   → Response: `{"error": "unauthorized"}`

3. `action == "replay"` without an explicit `replay_flow_id` from a prior
   captured session — refuses blind replay attacks.

4. CA certificate installation on a production system (detected via hostname
   matching known prod patterns) without a second confirmation flag
   `"prod_cert_install_confirmed": true`.

5. Any `listen_host` other than `127.0.0.1` or `::1` unless the engagement
   brief explicitly permits network-accessible proxy (corporate MITM scenario).

**Logging:** every refused action is written to
`/srv/dr/engagements/<id>/auth-refusals.log` with timestamp and reason.

---

## Handoff

| Next Agent | Condition | Payload |
|---|---|---|
| `sqli-surgeon` | After export, HAR contains POST/GET params | `har_file` path + `endpoints_file` |
| `xss-specter` | After export, HAR contains reflected input in responses | `har_file` path |
| `fingerprint-forge` | TLS fingerprint mismatch detected in proxy log | `target_scope` + `ca_cert_path` |
| `recon-specialist` | New hosts discovered in proxy traffic beyond initial scope | list of new hosts (for scope expansion approval) |
| `auth-auditor` | Session cookies / auth tokens visible in HAR | `har_file` with cookie extraction hint |
