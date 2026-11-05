# Proxy Rotation — Management and Integration for Scan/Attack Operations

> For authorized penetration testing, red team operations, and security research only. OPSEC guidance included per proxy type.

---

## Overview

Proxy rotation distributes outbound HTTP/S traffic across many IP addresses to avoid rate limits, IP bans, and geographic restrictions. This skill covers proxy types, rotation strategies, sourcing, tool integration, health checking, and the OPSEC reality of each approach.

---

## 1. Proxy Types

### 1.1 HTTP Proxy

```
Client → HTTP Proxy → Target
```

- Handles HTTP and HTTPS (via CONNECT tunnel for TLS)
- Most common type; supported by virtually all tools
- URL format: `http://user:pass@host:port`
- No UDP support

### 1.2 SOCKS4 Proxy

```
Client → SOCKS4 Proxy → Target (TCP only)
```

- Lower-level than HTTP proxy; works for any TCP application
- No authentication in SOCKS4 (auth added in SOCKS4a)
- SOCKS4a adds remote DNS resolution (target hostname resolved by proxy)
- URL format: `socks4://host:port` or `socks4a://host:port`

### 1.3 SOCKS5 Proxy

```
Client → SOCKS5 Proxy → Target (TCP + UDP)
```

- Supports username/password authentication
- Supports UDP (useful for DNS, some protocols)
- Remote DNS resolution supported
- URL format: `socks5://user:pass@host:port`
- **Preferred type** for most pentesting tools (supports SSH tunnels, Tor, ligolo)

### 1.4 Comparison Matrix

| Feature | HTTP | SOCKS4 | SOCKS5 |
|---|---|---|---|
| Authentication | Basic/Digest | None | Username/Password |
| UDP support | No | No | Yes |
| DNS leak risk | Medium | High | Low (with remote DNS) |
| Tool compat | Highest | Medium | High |
| TLS tunneling | Via CONNECT | Via CONNECT | Via CONNECT |
| Tor compatible | No | No | Yes (127.0.0.1:9050) |

---

## 2. Rotation Strategies

### 2.1 Round-Robin

Cycles through proxies in order — every Nth request uses proxy[N % len(proxies)].

```python
import itertools
import threading

class RoundRobinPool:
    def __init__(self, proxies: list[str]):
        self._cycle = itertools.cycle(proxies)
        self._lock = threading.Lock()

    def next(self) -> str:
        with self._lock:
            return next(self._cycle)
```

**When to use:** Equal load distribution, all proxies are known-healthy, no per-request failures.

### 2.2 Failure-Based Rotation (with Blacklist)

Marks proxies as failed on timeout/ban, removes them from rotation until cool-down.

```python
import time
import random
import threading

class FailoverPool:
    def __init__(self, proxies: list[str], cooldown_seconds: int = 300):
        self._all = proxies
        self._active: list[str] = list(proxies)
        self._failed: dict[str, float] = {}  # proxy -> fail timestamp
        self._cooldown = cooldown_seconds
        self._lock = threading.Lock()

    def get(self) -> str | None:
        with self._lock:
            # Restore proxies whose cooldown has expired
            now = time.monotonic()
            restored = [p for p, t in self._failed.items() if now - t > self._cooldown]
            for p in restored:
                self._active.append(p)
                del self._failed[p]

            if not self._active:
                return None  # all proxies failed
            return random.choice(self._active)

    def mark_failed(self, proxy: str):
        with self._lock:
            if proxy in self._active:
                self._active.remove(proxy)
            self._failed[proxy] = time.monotonic()

    def mark_banned(self, proxy: str):
        """Longer cooldown for banned proxies."""
        with self._lock:
            if proxy in self._active:
                self._active.remove(proxy)
            self._failed[proxy] = time.monotonic() - self._cooldown + 3600  # 1h cooldown
```

**When to use:** Residential proxy pools, scanning where bans are expected.

### 2.3 Time-Based Rotation

Rotate to a new proxy every N seconds or N requests, regardless of failure state.

```python
import time

class TimedRotation:
    def __init__(self, proxies: list[str], interval_seconds: int = 60):
        self._proxies = proxies
        self._interval = interval_seconds
        self._idx = 0
        self._last_rotate = time.monotonic()
        self._lock = threading.Lock()

    def get(self) -> str:
        with self._lock:
            if time.monotonic() - self._last_rotate >= self._interval:
                self._idx = (self._idx + 1) % len(self._proxies)
                self._last_rotate = time.monotonic()
            return self._proxies[self._idx]
```

**When to use:** Targets that fingerprint session duration; maintaining cover on long operations.

### 2.4 Weighted Rotation

Assigns higher weight to faster or more reliable proxies.

```python
import random

class WeightedPool:
    def __init__(self, proxies: list[tuple[str, float]]):
        # proxies = [("http://host:port", 3.0), ("http://host2:port", 1.0)]
        self._proxies = [p for p, _ in proxies]
        self._weights = [w for _, w in proxies]

    def get(self) -> str:
        return random.choices(self._proxies, weights=self._weights, k=1)[0]
```

---

## 3. Proxy Sources

### 3.1 Residential Proxies

- Real ISP-assigned IPs from real users; hardest to block
- Sources: BrightData, Oxylabs, Smartproxy, IPRoyal, Webshare (residential tier)
- Price: $5–20/GB
- **OPSEC:** Highest legitimacy; hard to distinguish from real user traffic. Preferred for targets with strong bot detection (Cloudflare, Akamai, PerimeterX)

### 3.2 Datacenter Proxies

- IPs hosted in cloud/DC ranges (AWS, DigitalOcean, Hetzner, OVH)
- Cheap: $0.5–3/GB or shared pools for $10–50/month
- **OPSEC:** Easily flagged by ASN; many DC IP ranges are pre-blocked by Cloudflare. Use only when target has weak bot detection.
- Sources: Webshare (DC tier), Proxy-Cheap, ProxyScrape (free but low quality)

### 3.3 TOR Network

```bash
# Install and run Tor
apt install tor
systemctl start tor

# Default SOCKS5 proxy: 127.0.0.1:9050
# New circuit (new IP) every ~10 minutes automatically

# Force new circuit via control port
echo -e 'AUTHENTICATE ""\nSIGNAL NEWNYM\nQUIT' | nc 127.0.0.1 9051
```

- **OPSEC:** Tor exit nodes are publicly listed — any target that checks against Tor exit lists (dan.me.uk/torlist) will block them. Good for research, not for evasion against security-aware targets.
- Slow: ~5–20 KB/s per circuit

### 3.4 Free Proxy Lists (Low Quality)

```bash
# Sources — heavily filtered public lists
# https://free-proxy-list.net/
# https://www.proxyscrape.com/free-proxy-list
# https://github.com/TheSpeedX/PROXY-List

# Download and test (see health checking below)
curl -s https://raw.githubusercontent.com/TheSpeedX/PROXY-List/master/http.txt -o proxies.txt
```

- **OPSEC:** Extremely poor. Most are honeypots or already blocked. Use only for CTF/non-sensitive work. Never for operations requiring actual anonymity.

### 3.5 Rotating Proxy Services (API-based)

Services that handle rotation automatically — you connect to a single endpoint, they rotate IPs per request or per session.

```bash
# BrightData residential rotating (example)
export PROXY="http://brd-customer-CUST_ID-zone-zone_name:password@zproxy.lum-superproxy.io:22225"

# Smartproxy rotating
export PROXY="http://user:pass@gate.smartproxy.com:7000"

# Usage — every request gets a different IP automatically
curl --proxy "$PROXY" https://api.ipify.org
```

---

## 4. Tool Integration

### 4.1 sqlmap

```bash
# Single proxy
sqlmap -u "http://target.com/?id=1" --proxy="http://127.0.0.1:8080"

# Random rotation from file (sqlmap built-in)
sqlmap -u "http://target.com/?id=1" --proxy-file=proxies.txt --randomize=1

# With Tor
sqlmap -u "http://target.com/?id=1" --tor --tor-type=SOCKS5 --check-tor

# Slow/stealth mode with proxy
sqlmap -u "http://target.com/?id=1" --proxy="http://127.0.0.1:8080" \
  --delay=2 --time-sec=10 --level=1 --risk=1
```

### 4.2 Python requests

```python
import requests

# Single proxy
proxies = {
    "http":  "http://user:pass@host:port",
    "https": "http://user:pass@host:port",
}
resp = requests.get("https://target.com/", proxies=proxies, timeout=15)

# SOCKS5 (requires: pip install requests[socks])
proxies = {
    "http":  "socks5://user:pass@host:port",
    "https": "socks5://user:pass@host:port",
}

# SOCKS5 with remote DNS (prevents DNS leaks)
proxies = {
    "http":  "socks5h://user:pass@host:port",
    "https": "socks5h://user:pass@host:port",
}
```

### 4.3 curl

```bash
# HTTP proxy
curl --proxy "http://user:pass@host:port" https://target.com/

# SOCKS5
curl --proxy "socks5://user:pass@host:port" https://target.com/

# SOCKS5 with remote DNS
curl --proxy "socks5h://user:pass@host:port" https://target.com/

# Environment variable (picked up by curl and many other tools)
export http_proxy="http://user:pass@host:port"
export https_proxy="http://user:pass@host:port"
curl https://target.com/

# Rotate proxies via shell loop
while IFS= read -r proxy; do
  curl --proxy "$proxy" --max-time 10 -s https://api.ipify.org && echo " $proxy"
done < proxies.txt
```

### 4.4 Go — http.Transport with Proxy

```go
import (
    "net/http"
    "net/url"
    "golang.org/x/net/proxy"
)

// HTTP proxy
proxyURL, _ := url.Parse("http://user:pass@host:port")
transport := &http.Transport{Proxy: http.ProxyURL(proxyURL)}
client := &http.Client{Transport: transport}

// SOCKS5 proxy
dialer, _ := proxy.SOCKS5("tcp", "host:port",
    &proxy.Auth{User: "user", Password: "pass"}, proxy.Direct)
transport2 := &http.Transport{Dial: dialer.Dial}
client2 := &http.Client{Transport: transport2}
```

### 4.5 proxychains4

```bash
# Config: /etc/proxychains4.conf
# Change to dynamic_chain for automatic failover
dynamic_chain

[ProxyList]
socks5 127.0.0.1 9050    # Tor
http   host1    8080
socks5 host2    1080 user pass

# Usage — prepend to any command
proxychains4 nmap -sT -Pn target.com
proxychains4 sqlmap -u "http://target.com/?id=1"
proxychains4 curl https://target.com/
```

### 4.6 Nuclei

```bash
# Single proxy
nuclei -u https://target.com -proxy http://127.0.0.1:8080

# Rotate with a local rotator on :8080
nuclei -u https://target.com -proxy http://127.0.0.1:8080 -rate-limit 5
```

---

## 5. Health Checking Proxies

### 5.1 Python Concurrent Health Checker

```python
import requests
import concurrent.futures
import time

CHECK_URL = "https://api.ipify.org"  # returns your public IP as plain text
TIMEOUT = 10

def check_proxy(proxy: str) -> tuple[str, bool, float, str]:
    proxies = {"http": proxy, "https": proxy}
    start = time.monotonic()
    try:
        resp = requests.get(CHECK_URL, proxies=proxies, timeout=TIMEOUT)
        elapsed = time.monotonic() - start
        if resp.status_code == 200:
            return proxy, True, elapsed, resp.text.strip()
    except Exception:
        pass
    return proxy, False, 0.0, ""

def health_check(proxy_file: str, workers: int = 50) -> list[str]:
    with open(proxy_file) as f:
        proxies = [line.strip() for line in f if line.strip()]

    working = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as ex:
        futures = {ex.submit(check_proxy, p): p for p in proxies}
        for future in concurrent.futures.as_completed(futures):
            proxy, ok, elapsed, ip = future.result()
            if ok:
                print(f"[OK] {proxy} → {ip} ({elapsed:.2f}s)")
                working.append(proxy)
            else:
                print(f"[DEAD] {proxy}")

    with open("working_proxies.txt", "w") as f:
        f.write("\n".join(working))
    print(f"\n{len(working)}/{len(proxies)} proxies working")
    return working

if __name__ == "__main__":
    health_check("proxies.txt", workers=100)
```

### 5.2 Go Health Checker

```go
package main

import (
	"bufio"
	"fmt"
	"net/http"
	"net/url"
	"os"
	"sync"
	"time"
)

func checkProxy(proxyURL string, wg *sync.WaitGroup, mu *sync.Mutex, working *[]string) {
	defer wg.Done()
	proxy, err := url.Parse(proxyURL)
	if err != nil {
		return
	}
	client := &http.Client{
		Transport: &http.Transport{Proxy: http.ProxyURL(proxy)},
		Timeout:   10 * time.Second,
	}
	resp, err := client.Get("https://api.ipify.org")
	if err != nil || resp.StatusCode != 200 {
		return
	}
	defer resp.Body.Close()
	mu.Lock()
	*working = append(*working, proxyURL)
	fmt.Printf("[OK] %s\n", proxyURL)
	mu.Unlock()
}

func main() {
	f, _ := os.Open("proxies.txt")
	scanner := bufio.NewScanner(f)
	var proxies []string
	for scanner.Scan() {
		if p := scanner.Text(); p != "" {
			proxies = append(proxies, p)
		}
	}
	f.Close()

	var wg sync.WaitGroup
	var mu sync.Mutex
	var working []string
	sem := make(chan struct{}, 100) // 100 concurrent

	for _, p := range proxies {
		wg.Add(1)
		sem <- struct{}{}
		go func(proxy string) {
			defer func() { <-sem }()
			checkProxy(proxy, &wg, &mu, &working)
		}(p)
	}
	wg.Wait()

	out, _ := os.Create("working_proxies.txt")
	for _, p := range working {
		fmt.Fprintln(out, p)
	}
	out.Close()
	fmt.Printf("%d/%d working\n", len(working), len(proxies))
}
```

---

## 6. Local Rotating Proxy Server

A local HTTP proxy that accepts connections and forwards through a pool — lets tools that only support a single proxy use rotation transparently.

```python
#!/usr/bin/env python3
"""local_rotator.py — HTTP CONNECT proxy that rotates upstream proxies"""
import socket, threading, random, time

UPSTREAM_PROXIES = [
    "http://user:pass@host1:port",
    "socks5://user:pass@host2:port",
    # ...
]
LISTEN_PORT = 8080

def get_proxy():
    return random.choice(UPSTREAM_PROXIES)

def handle(client_sock):
    try:
        data = client_sock.recv(4096)
        if not data:
            return
        first_line = data.split(b'\r\n')[0].decode()
        method, target, _ = first_line.split(' ')
        host, port = (target.split(':') + ['80'])[:2]
        port = int(port)

        upstream = get_proxy()
        # For simplicity, connect directly here (full implementation would chain through upstream)
        # In production, use mitmproxy or a proper rotating proxy library
        remote = socket.create_connection((host, port), timeout=15)
        if method == 'CONNECT':
            client_sock.sendall(b'HTTP/1.1 200 Connection established\r\n\r\n')
        else:
            remote.sendall(data)
        # Bidirectional relay
        def relay(src, dst):
            try:
                while chunk := src.recv(4096):
                    dst.sendall(chunk)
            except: pass
        t1 = threading.Thread(target=relay, args=(client_sock, remote), daemon=True)
        t2 = threading.Thread(target=relay, args=(remote, client_sock), daemon=True)
        t1.start(); t2.start()
        t1.join(); t2.join()
    except Exception:
        pass
    finally:
        client_sock.close()

server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('127.0.0.1', LISTEN_PORT))
server.listen(100)
print(f"Rotating proxy listening on 127.0.0.1:{LISTEN_PORT}")
while True:
    client, _ = server.accept()
    threading.Thread(target=handle, args=(client,), daemon=True).start()
```

For production use, prefer **mitmproxy** with an upstream proxy addon:

```python
# upstream_rotator.py — mitmproxy addon
import random
PROXIES = ["http://host1:port", "http://host2:port"]

def proxy_auth(flow):
    flow.server_conn.via = random.choice(PROXIES)
```

```bash
mitmdump -p 8080 -s upstream_rotator.py
```

---

## 7. OPSEC Considerations Per Proxy Type

| Proxy Type | Anonymity | Ban Risk | DNS Leak Risk | Speed | Cost |
|---|---|---|---|---|---|
| Residential rotating | Very High | Very Low | Low (with socks5h) | Medium | High |
| Datacenter rotating | Low | High | Medium | High | Low |
| TOR | High (but detectable) | High (exit lists) | Low | Very Low | Free |
| Free lists | None | Very High | High | Very Low | Free |
| Your own VPS | None (linked to you) | Low | None | High | Medium |

**Key OPSEC rules:**

1. **Never mix proxy types in a single operation** — correlating requests from residential + datacenter IPs reveals the pattern.
2. **Always use `socks5h://` (remote DNS)** — `socks5://` resolves DNS locally, leaking the target hostname to your ISP.
3. **Residential proxies for WAF-protected targets** — datacenter IPs are blocked by ASN in Cloudflare/Akamai at the edge.
4. **Rotate User-Agent along with IP** — IP rotation alone is insufficient; fingerprinting uses UA + TLS JA3 + accept headers together.
5. **TOR exit nodes are public** — check dan.me.uk/torlist before using Tor for OSEC-sensitive operations.
6. **Log nothing you wouldn't want disclosed** — proxy provider logs exist; residential proxies log usage by terms of service.

---

## 8. Integration with Common Attack Tools

### Comprehensive Proxy Config Reference

```bash
# nmap (no native proxy — route via proxychains)
proxychains4 nmap -sT -Pn -p 80,443,8080 target.com

# ffuf
ffuf -u https://target.com/FUZZ -w wordlist.txt -x http://127.0.0.1:8080

# gobuster
gobuster dir -u https://target.com/ -w wordlist.txt --proxy http://127.0.0.1:8080

# nikto
nikto -h https://target.com -useproxy http://127.0.0.1:8080

# hydra
hydra -L users.txt -P passwords.txt target.com http-post-form \
  "/login:user=^USER^&pass=^PASS^:Invalid" \
  -o hits.txt -e nsr

# wfuzz
wfuzz -p 127.0.0.1:8080 -c -w wordlist.txt https://target.com/FUZZ

# httpx (for recon)
cat domains.txt | httpx -proxy http://127.0.0.1:8080

# nuclei
nuclei -l targets.txt -proxy http://127.0.0.1:8080 -rate-limit 10

# sqlmap
sqlmap -u "https://target.com/?id=1" --proxy http://127.0.0.1:8080 --random-agent

# curl — all at once via env
export HTTP_PROXY=http://user:pass@host:port
export HTTPS_PROXY=http://user:pass@host:port
export NO_PROXY=localhost,127.0.0.1
```

---

## References

- BrightData (residential): https://brightdata.com
- Smartproxy: https://smartproxy.com
- ProxyScrape free lists: https://proxyscrape.com/free-proxy-list
- Tor Project: https://www.torproject.org
- mitmproxy proxy chain docs: https://docs.mitmproxy.org/stable/concepts-modes/#upstream-proxy
- proxychains4: https://github.com/haad/proxychains
- golang.org/x/net/proxy (SOCKS5): https://pkg.go.dev/golang.org/x/net/proxy
