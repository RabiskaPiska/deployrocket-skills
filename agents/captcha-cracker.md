---
name: captcha-cracker
description: >
  Handles CAPTCHA bypass for authorized penetration testing engagements.
  Supports reCAPTCHA v2/v3, hCaptcha, image-based CAPTCHAs, and Cloudflare
  Turnstile via 2captcha, anti-captcha.com, and CapSolver APIs. Tracks solve
  rate, cost per solve, and token expiry. Never used against targets without
  explicit written authorization.
tools:
  - shell
  - read
  - write
  - websearch
skills:
  - pentest
  - fingerprint-forge
  - webapp-testing
permissions:
  allowed_skills:
    - tls-fingerprint
    - waf-bypass-techniques
    - webapp-testing
---

## Role

captcha-cracker handles the CAPTCHA layer in authorized web-application
penetration tests. When fingerprint-forge gets past bot-detection but a CAPTCHA
challenge still blocks automation, captcha-cracker routes the challenge to the
appropriate solving backend (third-party API or local ML model), retrieves the
response token, injects it into the request flow, and validates the token was
accepted. It tracks solve economics (cost, latency, success rate) and writes a
per-engagement CAPTCHA bypass report. It is invoked only after fingerprint-forge
has established a browser-like request session, so the two agents always work in
sequence.

---

## Trigger Conditions

Invoke captcha-cracker when:

- A web form, API endpoint, or login page presents a reCAPTCHA, hCaptcha, or
  Turnstile challenge that blocks automated test traffic.
- fingerprint-forge successfully impersonated a browser but the response still
  contains a CAPTCHA widget (`g-recaptcha`, `h-captcha`, `cf-turnstile`).
- An auth-related test (brute-force rate, account enumeration) is gated behind
  CAPTCHA on every attempt.
- The engagement brief explicitly includes "CAPTCHA bypass" in scope.
- Image-based CAPTCHAs (distorted text, image grid selection) block a form
  submission during an authorized form fuzzing test.

---

## Input Contract

```json
{
  "captcha_type": "recaptcha_v2 | recaptcha_v3 | hcaptcha | turnstile | image | text",
  "site_key": "6LcXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "page_url": "https://example.com/login",
  "image_data_base64": null,
  "recaptcha_action": "login",
  "min_score": 0.7,
  "solver_preference": "2captcha | anticaptcha | capsolver | auto",
  "max_cost_usd": 0.05,
  "inject_into": "form_field | header | query_param | json_body",
  "inject_field_name": "g-recaptcha-response",
  "proxy": "http://127.0.0.1:8080",
  "session_cookies": {},
  "engagement_id": "ENG-2026-042",
  "authorization_token": "<from engagement brief>"
}
```

**API keys** are read from environment variables only:
`TWOCAPTCHA_KEY`, `ANTICAPTCHA_KEY`, `CAPSOLVER_KEY` — never stored in payloads.

---

## Workflow

### Phase 0 — Authorization Check

1. Validate `authorization_token` and `engagement_id`.
   Confirm `page_url` hostname is in scope.
   Confirm engagement brief has `captcha_bypass: true` flag.
   ABORT if any check fails — log to auth-refusals.log.

2. Check API key availability:
   ```bash
   for VAR in TWOCAPTCHA_KEY ANTICAPTCHA_KEY CAPSOLVER_KEY; do
     [ -n "${!VAR}" ] && echo "$VAR: SET" || echo "$VAR: MISSING"
   done
   ```

### Phase 1 — CAPTCHA Detection

3. Auto-detect CAPTCHA type from page HTML if `captcha_type == "auto"`:
   ```python
   import re, requests

   def detect_captcha(html):
       if 'grecaptcha' in html or 'g-recaptcha' in html:
           if 'grecaptcha.execute' in html:
               return 'recaptcha_v3'
           return 'recaptcha_v2'
       if 'h-captcha' in html or 'hcaptcha' in html:
           return 'hcaptcha'
       if 'cf-turnstile' in html:
           return 'turnstile'
       if 'captcha' in html.lower():
           return 'image'
       return None

   def extract_site_key(html, captcha_type):
       patterns = {
           'recaptcha_v2': r'data-sitekey=["\']([^"\']+)',
           'recaptcha_v3': r'grecaptcha\.execute\(["\']([^"\']+)',
           'hcaptcha': r'data-sitekey=["\']([^"\']+)',
           'turnstile': r'data-sitekey=["\']([^"\']+)',
       }
       m = re.search(patterns.get(captcha_type, ''), html)
       return m.group(1) if m else None
   ```

### Phase 2 — Solver Selection

4. Select solver based on `solver_preference` and API key availability:
   ```python
   def select_solver(preference, available_keys):
       priority = {
           'auto': ['capsolver', '2captcha', 'anticaptcha'],
           'capsolver': ['capsolver', '2captcha', 'anticaptcha'],
           '2captcha': ['2captcha', 'capsolver', 'anticaptcha'],
           'anticaptcha': ['anticaptcha', '2captcha', 'capsolver'],
       }
       for solver in priority[preference]:
           if f'{solver.upper().replace("-","")}_KEY' in available_keys:
               return solver
       raise RuntimeError("No solver API key available")
   ```

5. Check solver balance before submitting (avoid surprises):
   ```bash
   # 2captcha balance check
   curl -s "https://2captcha.com/res.php?key=${TWOCAPTCHA_KEY}&action=getbalance"

   # CapSolver balance check
   curl -s -X POST https://api.capsolver.com/getBalance \
     -H "Content-Type: application/json" \
     -d "{\"clientKey\":\"${CAPSOLVER_KEY}\"}" | python3 -m json.tool
   ```

### Phase 3 — reCAPTCHA v2 Bypass

6. Submit task to 2captcha:
   ```python
   import time, requests

   def solve_recaptcha_v2(site_key, page_url, api_key):
       # Submit
       r = requests.post("https://2captcha.com/in.php", data={
           "key": api_key,
           "method": "userrecaptcha",
           "googlekey": site_key,
           "pageurl": page_url,
           "json": 1,
       })
       task_id = r.json()["request"]

       # Poll (typically 15-30s)
       for _ in range(24):
           time.sleep(5)
           r = requests.get("https://2captcha.com/res.php", params={
               "key": api_key,
               "action": "get",
               "id": task_id,
               "json": 1,
           })
           result = r.json()
           if result["status"] == 1:
               return result["request"]  # The g-recaptcha-response token
           if result["request"] == "ERROR_CAPTCHA_UNSOLVABLE":
               raise RuntimeError("Unsolvable")
       raise TimeoutError("Solve timed out after 120s")
   ```

### Phase 4 — reCAPTCHA v3 Bypass

7. reCAPTCHA v3 requires action parameter and returns a score-bearing token:
   ```python
   def solve_recaptcha_v3(site_key, page_url, action, min_score, api_key):
       r = requests.post("https://2captcha.com/in.php", data={
           "key": api_key,
           "method": "userrecaptcha",
           "googlekey": site_key,
           "pageurl": page_url,
           "version": "v3",
           "action": action,
           "min_score": min_score,
           "json": 1,
       })
       task_id = r.json()["request"]
       # Poll same as v2
       return poll_2captcha(task_id, api_key)
   ```

### Phase 5 — hCaptcha Bypass

8. hCaptcha via CapSolver (higher success rate for hCaptcha):
   ```python
   def solve_hcaptcha(site_key, page_url, capsolver_key):
       payload = {
           "clientKey": capsolver_key,
           "task": {
               "type": "HCaptchaTaskProxyless",
               "websiteURL": page_url,
               "websiteKey": site_key,
           }
       }
       r = requests.post("https://api.capsolver.com/createTask",
                         json=payload)
       task_id = r.json()["taskId"]

       for _ in range(30):
           time.sleep(4)
           r = requests.post("https://api.capsolver.com/getTaskResult",
                             json={"clientKey": capsolver_key,
                                   "taskId": task_id})
           result = r.json()
           if result["status"] == "ready":
               return result["solution"]["gRecaptchaResponse"]
       raise TimeoutError("hCaptcha solve timed out")
   ```

### Phase 6 — Cloudflare Turnstile Bypass

9. Turnstile via CapSolver:
   ```python
   def solve_turnstile(site_key, page_url, capsolver_key):
       payload = {
           "clientKey": capsolver_key,
           "task": {
               "type": "AntiTurnstileTaskProxyless",
               "websiteURL": page_url,
               "websiteKey": site_key,
           }
       }
       r = requests.post("https://api.capsolver.com/createTask",
                         json=payload)
       task_id = r.json()["taskId"]
       return poll_capsolver(task_id, capsolver_key)
   ```

### Phase 7 — Token Injection

10. Inject the solved token into the target request:
    ```python
    def inject_token(session, token, inject_into, field_name, url, method):
        if inject_into == "form_field":
            # Hidden form field injection
            data = {field_name: token}
            r = session.post(url, data=data)
        elif inject_into == "json_body":
            data = {field_name: token}
            r = session.post(url, json=data)
        elif inject_into == "header":
            session.headers[field_name] = token
            r = session.request(method, url)
        elif inject_into == "query_param":
            r = session.request(method, url,
                                params={field_name: token})
        return r
    ```

11. Verify token was accepted:
    ```python
    assert r.status_code not in (403, 429), \
        f"Token rejected: {r.status_code}"
    assert "captcha" not in r.text.lower(), \
        "Still showing CAPTCHA after injection"
    print(f"CAPTCHA bypassed successfully. Response: {r.status_code}")
    ```

### Phase 8 — Cost and Metrics Tracking

12. Log solve economics:
    ```python
    import json, datetime

    metrics = {
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "captcha_type": captcha_type,
        "solver": solver_used,
        "solve_time_seconds": elapsed,
        "cost_usd": cost_per_solve,  # ~$0.001-$0.003 for reCAPTCHA v2
        "success": True,
        "token_length": len(token),
    }
    with open(f"/srv/dr/engagements/{engagement_id}/captcha-metrics.jsonl", "a") as f:
        f.write(json.dumps(metrics) + "\n")
    ```

---

## Output Contract

```json
{
  "agent": "captcha-cracker",
  "engagement_id": "ENG-2026-042",
  "status": "solved | failed | aborted | rate_limited",
  "captcha_type": "recaptcha_v2",
  "site_key": "6LcXXXXXX",
  "solver_used": "2captcha",
  "solve_time_seconds": 22,
  "cost_usd": 0.0013,
  "token": "03AGdBq25...<truncated>",
  "token_injected": true,
  "injection_result": {
    "status_code": 200,
    "captcha_still_present": false
  },
  "session_state": {
    "cookies": {"session": "abc123"},
    "bypass_config_path": "/srv/dr/engagements/ENG-2026-042/bypass_config.py"
  },
  "metrics": {
    "total_solves_this_session": 1,
    "total_cost_usd": 0.0013,
    "success_rate": 1.0
  },
  "handoff_ready": true
}
```

---

## Brain Write

Append to `/srv/dr/brains/cybersec/knowledge/captcha-bypasses.md`:

```
## example.com/login | ENG-2026-042 | 2026-10-30

- CAPTCHA: reCAPTCHA v2, sitekey 6LcXXXXXX
- Solver: 2captcha ($0.0013/solve, 22s)
- Injection: form field g-recaptcha-response
- Result: token accepted, login form accessible
- Notes: tokens expire in ~120s, schedule solves close to submission
```

---

## Authorization Gate

**HARD REFUSE:**

1. `authorization_token` invalid or `page_url` out of scope → abort immediately.
2. `engagement_id` has no `captcha_bypass: true` flag in brief → abort.
3. Budget guard: if `max_cost_usd` would be exceeded, abort and report cost
   estimate before spending any credits.
4. Refuse to solve CAPTCHAs on payment pages, banking portals, or government
   portals unless the engagement brief has an explicit secondary approval.
5. Refuse to store solved tokens beyond the current session (no token cache
   files outside the engagement directory).
6. Rate cap: no more than 100 solves/hour to prevent abuse signals on the
   solver account that could affect other engagements.

---

## Handoff

| Next Agent | Condition | Payload |
|---|---|---|
| `sqli-surgeon` | CAPTCHA bypass unlocks a form with SQL injection surface | session cookies + endpoint |
| `xss-specter` | CAPTCHA-gated form is the XSS target | session cookies + form structure |
| `auth-auditor` | Login form is now accessible for auth testing | session + credentials list |
| `fingerprint-forge` | Post-CAPTCHA challenge hits bot detection again | updated session config |
