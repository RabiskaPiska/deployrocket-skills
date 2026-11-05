# mcp-health-checker

## Purpose
Pre-flight and on-demand health check for every declared MCP server in the fleet. Fires before any MCP-dependent operation so the session knows which servers are alive, which are degraded, and what fallback to use.

## Trigger Conditions
- Session startup (auto, if any MCP tool is about to be called)
- User says: "check MCP status", "is MCP up", "health check", "MCP down?", "why is [tool] failing"
- Any MCP tool call returns an error or times out
- Prefix of a research or agentic task that relies on GitHub, Context7, Playwright, or the security hub

---

## Declared MCP Servers (fleet canonical list — Oct 2026)

| Alias | Server ID | Role |
|---|---|---|
| github-pat | `github` MCP (PAT-auth) | Repo reads, issue writes, search |
| context7 | `context7` MCP | Library / framework docs lookup |
| playwright | `playwright` MCP | Browser automation, screenshots |
| security-hub | `exa` + `tavily` + `jina` cluster | CVE lookup, web research, deep search |

---

## Health Check Procedure

### 1. GitHub PAT Server
```
CHECK: tools.github.get_latest_release({ owner: "anthropics", repo: "claude-code" })
PASS : returns JSON with tag_name field present
FAIL : timeout (>8 s), HTTP 401, HTTP 403, or error message
DEGRADED: returns but missing expected fields
```
**On FAIL:** use `tools.exa.web_search_exa` for GitHub-hosted content; use `tools.grep.searchGitHub` for code search. Log: "github-pat: DOWN — using exa fallback."

### 2. Context7
```
CHECK: tools.context7["resolve-library-id"]({ libraryName: "react", query: "hooks" })
PASS : returns items array with at least one entry
FAIL : timeout, empty result, or error
```
**On FAIL:** fall back to `jina.search_web` with explicit `site:` filter on the library's official docs domain. Log: "context7: DOWN — using jina direct-search fallback."

### 3. Playwright MCP
```
CHECK: tools.playwright.browser_navigate({ url: "about:blank" }) — or equivalent list_pages ping
PASS : returns page list or confirms navigation
FAIL : timeout, "no browser", or connection refused
```
**On FAIL:** skip screenshot-dependent verification steps; substitute curl/wget for HTTP checks; note in deliverable: "Screenshot verification skipped — Playwright unavailable." Log: "playwright: DOWN — skipping visual checks."

### 4. Security Hub (Exa + Tavily + Jina cluster)
Run all three simultaneously:
```
exa   : tools.exa.web_search_exa({ urls: ["https://nvd.nist.gov"] })
tavily: tools["tavily"].tavily_search({ query: "test" })
jina  : tools.jina.search_web({ query: ["test"] })
```
**PASS**: any one of the three responds.
**DEGRADED**: one or two respond — use the live ones, note which are down.
**FAIL**: all three timeout or error — fall back to `tools["parallel-search"].web_search` or direct `webfetch`. Log: "security-hub: ALL DOWN — using parallel-search fallback."

---

## Health Report Format

After running checks, emit a compact status block:

```
MCP HEALTH — <ISO timestamp>
  github-pat    : ✅ UP  (234 ms)
  context7      : ✅ UP  (411 ms)
  playwright    : ❌ DOWN — fallback: curl/wget for HTTP, no screenshots
  security-hub  : ⚠️  DEGRADED — exa✅ tavily❌ jina✅ — using exa+jina
```

Then immediately continue with the task using the listed fallbacks. Never block the session waiting for a downed server to recover — substitute and proceed.

---

## Fallback Matrix

| Server down | Primary fallback | Secondary fallback |
|---|---|---|
| github-pat | `exa.web_search_exa` + `grep.searchGitHub` | `parallel-search.web_search` + manual URL fetch |
| context7 | `jina.search_web` on official docs domain | `exa.web_fetch_exa` on known docs URL |
| playwright | HTTP-only verification, skip screenshots | Note "unverified visually" in deliverable |
| security-hub (all) | `parallel-search.web_search` | `webfetch` to nvd.nist.gov, github.com/advisories |

---

## Session-Startup Fast Path

If GodMode is active and the task is non-trivial, run the health check as a parallel `execute` block before the first substantive tool call — do not block the main thread. If all four pass, emit a single line: `MCP: all servers healthy` and proceed. Only emit the full table if at least one is degraded/down.

---

## Runbook: Common Failures

**"github: 401 / Bad credentials"**
→ PAT expired. Notify user: "GitHub PAT needs rotation — go to github.com/settings/tokens." Continue with exa fallback.

**"context7: empty items array"**
→ Library not indexed. Use `jina.search_web` with `"<library> docs site:<official-domain>"`.

**"playwright: ECONNREFUSED"**
→ Browser process not running. On VPS: `pkill chromium; sleep 2` — Playwright MCP restarts it on next call. On PC: restart OpenCode.

**"exa: 402 USAGE_EXCEEDED"**
→ Monthly credit exhausted. Switch to `tavily` + `jina` exclusively for the rest of the session. Do NOT retry exa.

---

## Notes
- Health checks are lightweight pings — do not use them for real data.
- Cache the result for the session duration; do not re-run the check on every tool call.
- If the user explicitly says "force re-check MCP", invalidate the cache and re-run.
