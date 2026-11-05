---
name: tool-dispatcher
description: Routes to the correct vault tool for any given task before execution begins. Trigger on: "which tool should I use for X", "what tool handles Y", before any tool call where the right tool is ambiguous, or when a previous tool call returned nothing useful. Includes a category routing table, vault URL structure, and fallback logic when no vault tool exists.
---

# Tool Dispatcher

## Trigger

Fire BEFORE any tool execution where the correct tool is not already known with certainty. Also fires reactively when a tool call returns empty, errors, or an obviously wrong result — that is a mis-route, not a tool failure.

## Instructions

### Phase 1 — Classify the Task Type

Map the task to one of these primary categories (pick the MOST specific match):

| Task Type | Examples |
|-----------|---------|
| CODE_READ | "find where X is defined", "what does this function do", "trace callers of Y" |
| CODE_WRITE | "add a feature", "fix this bug", "refactor this file" |
| CODE_SEARCH | "find all usages of X", "which files import Y", "grep for pattern Z" |
| FILE_OPS | "create file", "edit file", "read file", "list directory" |
| WEB_RESEARCH | "look up docs for X", "find CVE for Y", "check latest version of Z" |
| WEB_FETCH | "read this URL", "scrape this page", "get the content of this link" |
| SHELL_CMD | "run a build", "install a package", "run tests", "git operation" |
| BROWSER | "take a screenshot", "open this page", "click this button", "preview a file" |
| AGENT_SPAWN | "do this in parallel", "run this in the background", "delegate this subtask" |
| DESIGN | "generate an image", "create a video", "design a logo" |
| DATA | "query the database", "run a SQL query", "check Supabase" |

### Phase 2 — Route to the Correct Tool

**CODE_READ / CODE_SEARCH:**
- First choice: `grep` (pattern search across files)
- Second choice: `glob` (find files by name pattern)
- Third choice: `read` (open the specific file once located)
- Never use `shell` with `cat` / `head` / `grep` — the dedicated tools are faster and safer

**CODE_WRITE:**
- Existing file with targeted change: `edit` (oldString → newString, must be unique)
- New file or full replacement: `write`
- Never use `shell` with `echo`/`printf`/`heredoc`/`tee` to write files

**FILE_OPS:**
- Read: `read` tool (not `shell cat`)
- List directory: `read` on the directory path
- Create: `write`
- Edit: `edit`
- Find by pattern: `glob`

**WEB_RESEARCH:**
- First choice: `websearch` (broad query, snippets)
- Second choice: `execute` → `tools.exa.web_fetch_exa` (when you need page body, not snippets)
- Third choice: `webfetch` (single known URL, you just need one fact)
- Never trust training memory for: version numbers, CVEs, pricing, API changes, ToS

**WEB_FETCH (known URL):**
- `webfetch` for a single URL when only one fact is needed
- `execute` → `tools.exa.web_fetch_exa` when you need the full cleaned markdown

**SHELL_CMD:**
- Use `shell` ONLY for: git, npm/pip/composer installs, builds, tests, docker, dev servers
- All file operations stay on dedicated tools even inside shell-heavy sessions

**BROWSER:**
- `execute` → `tools.browser.preview` to show a file in the Review pane
- `execute` → `tools.playwright.*` for automated browser interaction
- `execute` → `tools["chrome-devtools"].*` for DevTools inspection

**AGENT_SPAWN:**
- `subagent` for delegating a bounded subtask to a specialist agent
- Pass a complete prompt — subagents start with fresh context
- Use `background: true` for independent parallel work

**DESIGN:**
- Images: `execute` → `tools.higgsfield.*` or `tools["media-studio"].*`
- Check media credit cost before spawning

**DATA:**
- `execute` → `tools.supabase.*` for Supabase queries
- `shell` with parameterized CLI calls for raw DB ops (never interpolate user input)

### Phase 3 — Vault URL Structure

When a tool is listed in the Code Mode catalog, reach it via `execute`:

```javascript
// Pattern: tools["namespace"]["tool-name"](input)
// Example:
const results = await tools.exa.web_fetch_exa({ urls: ["https://example.com"] });
```

To discover unlisted tools:
```javascript
const found = search({ query: "your tool description here", limit: 5 });
// Returns: { items: [{ path, description, signature }] }
```

Never guess a tool path. If `search()` returns nothing for a query, the tool does not exist in this vault — use fallback logic below.

### Phase 4 — Fallback Logic

When no vault tool fits:

1. **Compose from primitives** — combine `grep` + `read` + `edit` instead of reaching for a hypothetical specialist tool.
2. **Shell as last resort** — `shell` can do almost anything but costs more tokens and is harder to sandbox. Use it only when no dedicated tool exists.
3. **Subagent delegation** — if the task needs a capability this session cannot provide (e.g., GUI interaction, a different OS), spawn a subagent with the right context.
4. **Report the gap** — after the task completes, note which tool was missing and what it would have done; this feeds the vault improvement backlog.

### Anti-Patterns (never do these)

- `shell` with `cat file.txt` — use `read`
- `shell` with `grep pattern .` — use `grep`
- `shell` with `echo "..." > file` — use `write`
- `shell` with `find . -name "*.ts"` — use `glob`
- `shell` with `sed -i` — use `edit`
- Calling a tool twice with the same input after it returned nothing — re-classify first

## References

- Code Mode catalog: listed in the `execute` tool description in this session
- Related skills: `context-curator` (what to load), `fleet-monitor` (track execution), `godmode` (overall orchestration)
- Tool docs: see each tool's description in the `<tools>` block at session start

## Scripts

```javascript
// Discover tools matching a description (run inside execute)
const results = search({ query: "search github code", limit: 3 });
return results.items.map(t => ({ path: t.path, sig: t.signature }));
```

```bash
# Confirm a shell tool is available before using it
command -v rg && echo "ripgrep available" || echo "use grep tool instead"
```