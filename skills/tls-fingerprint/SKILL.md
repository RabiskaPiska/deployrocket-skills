---
name: tls-fingerprint
description: TLS/HTTP fingerprint impersonation for bot-detection bypass — JA3/JA4 analysis, curl_cffi browser presets, bogdanfinn/tls-client Go profiles, BrowserForge Bayesian header correlation, Camoufox/Patchright browser bridge, HTTP/2 SETTINGS spoofing, QUIC/HTTP3 transport-parameter impersonation. Use when a target blocks default Python/Go/Node HTTP clients via passive fingerprinting (Cloudflare, Akamai, DataDome, PerimeterX, Kasada), or when auditing TLS client-hello fingerprints for detection-surface analysis.
---

## Trigger

Fire when: a request is blocked before application data is served; the word "fingerprint", "JA3", "JA4", "Cloudflare challenge", "DataDome", "PerimeterX", "Kasada", "TLS impersonation", or "browser preset" appears; or a checker/scraper needs to match a real browser's TLS + HTTP/2 stack.

---

## The Fingerprint Stack (4 passive channels)

| Channel | What is leaked | Detection tool |
|---------|---------------|----------------|
| TLS ClientHello (JA3/JA4) | Cipher suite ordering, extensions, ALPN, ECH | Cloudflare, FoxIO JA4 |
| HTTP/2 SETTINGS frame | Window size, header table size, push flag, dependency weights | Akamai bot manager |
| HTTP header ordering | Accept-Language before Accept-Encoding, etc. | DataDome |
| QUIC transport params | Initial packet ordering, version negotiation | Akamai/Kasada QUIC gate |

---

## Tool Selection Matrix

| Stack | Primary pick | Install | When to use |
|-------|-------------|---------|-------------|
| Python async | `curl_cffi` | `pip install curl_cffi` | 47 free browser presets, HTTP/3, per-request proxy rotation |
| Python sync/async w/ header correlation | `hrequests` + `browserforge` | `pip install hrequests[all] browserforge[all]` | Bayesian TLS+header correlation prevents mismatch |
| Go (no CGO) | `Noooste/azuretls-client` | `go get github.com/Noooste/azuretls-client` | HTTP/2 SETTINGS + WINDOW_UPDATE + ALPN spoofing |
| Go (high-star, QUIC) | `enetx/surf` | `go get github.com/enetx/surf` | Full HTTP/3 QUIC fingerprinting, fluent builder |
| Go (FFI, many profiles) | `bogdanfinn/tls-client` | `go get github.com/bogdanfinn/tls-client` | ~76 browser profiles, HTTP/3 racing, WebSocket consistency |
| Browser-level automation | `daijro/camoufox` | `pip install "camoufox[geoip]"` | C++ patches for 20+ fingerprint traits, isolated JS world |
| Analysis/audit | `FoxIO-LLC/ja4` | `pip install ja4` | JA4+ suite, 11 fingerprint types, Zeek+Wireshark plugins |
| Self-hosted reflector | `North-web-dev/fpcheck` | self-host Go binary | Diff your client against reference browser byte-by-byte |

---

## Instructions

### 1. Diagnose current fingerprint
```bash
# Using fpcheck (self-hosted reflector)
curl -s https://your-fpcheck-instance/json | python -m json.tool

# Or use ja4 CLI against a pcap
ja4 -r capture.pcap --tls --http2
```

### 2. Python: curl_cffi with browser preset
```python
from curl_cffi import requests

# 47 built-in browser presets: chrome120, chrome124, firefox122, safari17_0, etc.
session = requests.Session(impersonate="chrome124")
resp = session.get("https://target.com/", proxies={"https": "http://proxy:port"})
```

### 3. Python: BrowserForge header correlation (prevents TLS-header mismatch)
```python
from browserforge.headers import HeaderGenerator
from curl_cffi import requests

gen = HeaderGenerator(browser="chrome", os="windows", locale=["en-US", "en"])
headers = gen.generate()

session = requests.Session(impersonate="chrome124")
session.headers.update(headers)
resp = session.get("https://target.com/")
```

### 4. Go: azuretls-client (pure Go, HTTP/2 SETTINGS spoofing)
```go
import (
    "github.com/Noooste/azuretls-client"
    "github.com/Noooste/fhttp"
)

session := azuretls.NewSession()
defer session.Close()

// Set HTTP/2 SETTINGS frame to match Chrome 124
session.SetH2Settings(azuretls.H2Settings{
    HeaderTableSize:      65536,
    MaxConcurrentStreams: 1000,
    InitialWindowSize:    6291456,
    MaxHeaderListSize:    262144,
})

req, _ := fhttp.NewRequest("GET", "https://target.com/", nil)
resp, err := session.Do(req)
```

### 5. Go: bogdanfinn/tls-client (large profile library)
```go
import (
    tls_client "github.com/bogdanfinn/tls-client"
    "github.com/bogdanfinn/tls-client/profiles"
)

options := []tls_client.HttpClientOption{
    tls_client.WithClientProfile(profiles.Chrome_124),
    tls_client.WithRandomTLSExtensionOrder(),
    tls_client.WithNotFollowRedirects(),
}

client, _ := tls_client.NewHttpClient(tls_client.NewNoopLogger(), options...)
req, _ := http.NewRequest("GET", "https://target.com/", nil)
resp, _ := client.Do(req)
```

### 6. Verify your fingerprint matches Chrome
```python
import json, requests as r
# Using browserleaks-style check or fpcheck
resp = session.get("https://tls.browserleaks.com/json")
fp = json.loads(resp.text)
print(fp["ja3_hash"])   # should match Chrome 124 reference
print(fp["ja4"])
```

---

## Anti-Detection Checklist

- [ ] JA3/JA4 hash matches the browser you are impersonating (use fpcheck or TLS reflector)
- [ ] HTTP/2 SETTINGS frame matches (HeaderTableSize, InitialWindowSize, MaxConcurrentStreams)
- [ ] Header order matches (Accept before Accept-Encoding for Chrome; opposite for Firefox)
- [ ] Accept-Language locale matches the proxy geo (US proxy → `en-US,en;q=0.9`)
- [ ] TLS extension ordering is randomized per request (use `WithRandomTLSExtensionOrder()`)
- [ ] User-Agent, sec-ch-ua, sec-ch-ua-platform are internally consistent
- [ ] No `X-Forwarded-For` leak from the proxy layer

---

## References

- `lexiforest/curl_cffi` — https://github.com/lexiforest/curl_cffi (6.8k stars, MIT, 2026-10-26)
- `bogdanfinn/tls-client` — https://github.com/bogdanfinn/tls-client (MIT, v1.16.0 Sep 2026)
- `daijro/hrequests` — https://github.com/daijro/hrequests (MIT, v0.9.2)
- `daijro/browserforge` — https://github.com/daijro/browserforge (MIT, Bayesian Apify-trained)
- `enetx/surf` — https://github.com/enetx/surf (1.8k stars, MIT)
- `Noooste/azuretls-client` — https://github.com/Noooste/azuretls-client (MIT, v1.13.2)
- `refraction-networking/utls` — https://github.com/refraction-networking/utls (3k+ stars, BSD-3, v1.8.2)
- `daijro/camoufox` — https://github.com/daijro/camoufox (12.8k stars, MPL-2.0)
- `FoxIO-LLC/ja4` — https://github.com/FoxIO-LLC/ja4 (2.1k stars, FoxIO License, v1.0.1)
- Source report: `fingerprint-forge-deep-research-2026-10-27.md`