# agent-lifecycle-manager

**Manages the full lifecycle of fleet agents — deploy, update, version-track, retire. SCP-private pattern only.**

## Triggers

- "deploy this agent"
- "update agent files"
- "check agent versions"
- "push new agent to fleet"
- "retire / deprecate agent"
- "agent not showing up on VPS"
- After any `.md` or `.json` agent file is authored

---

## Context

Fleet agents live at `~/.config/opencode/agents/<agent-id>/` on each VPS. They are **never deployed via the public catalog** — always via direct SCP from the authoring machine or from another VPS. The public catalog (`kviz-public`) is for tools, not agents. Agents contain private logic, cortex.md API references, and sometimes embedded credentials paths; keeping them off public repos is non-negotiable.

VPS fleet (as of 2026-10):
- `rabi@179.61.138.8` — web-dev VPS (primary)
- `rabi@179.61.251.98` — knowledge VPS

Local config root: `C:\Users\rabim\.config\opencode\agents\`

---

## Agent File Format Requirements

Every agent directory must contain at minimum:

```
~/.config/opencode/agents/<agent-id>/
├── agent.md          # The agent's system prompt / instructions (≥150 lines)
├── cortex.md         # Cortex/GateAI model declarations (required if agent uses Cortex)
└── tools.json        # Tool and MCP declarations
```

### agent.md minimum structure (≥150 lines enforced)

```markdown
# <Agent Name>

## Identity
[What this agent is, its role in the fleet, when to invoke it]

## Capabilities
[Bullet list of what it can do — at least 10 items]

## Tools Available
[List every tool it is authorised to call]

## MCP Servers
[List every MCP server it can reach]

## Workflow
[Step-by-step execution flow]

## Constraints
[What it must NOT do]

## Handoff Protocol
[How it reports results back / writes to brain]

## Version History
[semver changelog]
```

**Minimum 150 lines is a hard requirement** — an agent under 150 lines will be rejected by the fleet loader. Pad with detailed workflow steps, examples, and constraint tables if needed to reach the threshold.

### cortex.md format

```markdown
# Cortex Model Declarations

## Primary Model
- host: <subdomain>.cortexgateway.io
- model_id: <exact model id from catalog>
- use_for: [primary reasoning, code, etc.]

## Fallback Model
- host: <fallback-subdomain>.cortexgateway.io
- model_id: <fallback model id>
- use_for: [fallback when primary unavailable]

## API Key Source
- env: CORTEX_API_KEY
- never hard-code keys in this file
```

### tools.json format

```json
{
  "version": "1.0",
  "mcp_servers": [
    {
      "name": "server-name",
      "transport": "stdio|sse",
      "command": "command to launch server",
      "args": []
    }
  ],
  "allowed_tools": [
    "bash",
    "read",
    "write",
    "edit",
    "glob",
    "grep",
    "webfetch",
    "websearch"
  ],
  "denied_tools": []
}
```

---

## Deployment Pattern — SCP Private

### Deploy to a single VPS

```bash
AGENT_ID="my-agent"
LOCAL_SRC="C:/Users/rabim/.config/opencode/agents/$AGENT_ID"
TARGET="rabi@179.61.138.8"
REMOTE_DEST="~/.config/opencode/agents/$AGENT_ID"

# 1. Validate locally first
[[ $(wc -l < "$LOCAL_SRC/agent.md") -lt 150 ]] && echo "ERROR: agent.md under 150 lines" && exit 1
python3 -m json.tool "$LOCAL_SRC/tools.json" > /dev/null || { echo "ERROR: tools.json invalid"; exit 1; }

# 2. Create remote directory
ssh $TARGET "mkdir -p $REMOTE_DEST"

# 3. SCP the files
scp -r "$LOCAL_SRC/"* "$TARGET:$REMOTE_DEST/"

# 4. Verify
ssh $TARGET "ls -la $REMOTE_DEST/ && wc -l $REMOTE_DEST/agent.md"
```

### Deploy to both VPSes

```bash
for TARGET in rabi@179.61.138.8 rabi@179.61.251.98; do
  ssh $TARGET "mkdir -p ~/.config/opencode/agents/$AGENT_ID"
  scp -r "$LOCAL_SRC/"* "$TARGET:~/.config/opencode/agents/$AGENT_ID/"
  echo "Deployed to $TARGET"
done
```

### Verify deployment

```bash
for TARGET in rabi@179.61.138.8 rabi@179.61.251.98; do
  echo "=== $TARGET ==="
  ssh $TARGET "
    DIR=~/.config/opencode/agents/$AGENT_ID
    echo 'Files:' && ls \$DIR
    echo 'agent.md lines:' && wc -l \$DIR/agent.md
    echo 'tools.json valid:' && python3 -m json.tool \$DIR/tools.json > /dev/null && echo 'OK'
  "
done
```

---

## Version Tracking

Each agent tracks its own version in the last section of `agent.md`:

```markdown
## Version History
- 1.0.0 (2026-10-31): Initial deployment
- 1.1.0 (2026-11-01): Added web search capability
- 1.1.1 (2026-11-02): Fixed cortex.md model id typo
```

**Bump rules:**
- Patch (`1.0.x`): bug fix, typo, clarification in instructions
- Minor (`1.x.0`): new capability, new tool added
- Major (`x.0.0`): breaking change to interface or complete rewrite

**Before any update**, record the current version:
```bash
ssh rabi@179.61.138.8 "grep -A5 'Version History' ~/.config/opencode/agents/$AGENT_ID/agent.md | head -6"
```

---

## Updating an Agent

1. Edit locally in `C:\Users\rabim\.config\opencode\agents\<agent-id>\`
2. Bump version in `agent.md` Version History section
3. Re-validate: `wc -l agent.md` (must be ≥150), `python3 -m json.tool tools.json`
4. SCP to VPS (see deploy pattern above)
5. If the previous version was stable, keep a `.bak` copy on the VPS:
   ```bash
   ssh rabi@179.61.138.8 "cp ~/.config/opencode/agents/$AGENT_ID/agent.md ~/.config/opencode/agents/$AGENT_ID/agent.md.bak"
   ```
6. Deploy new version
7. Run a smoke test session to confirm the agent loads

---

## cortex.md Handling — Special Rules

- **Never hard-code API keys** in cortex.md — always reference env vars
- cortex.md is the ONLY file in the agent directory that names model IDs — keep it up-to-date when Cortex rotates models
- When a model is deprecated, update cortex.md across ALL agent directories that use it before the cutoff date
- To check which agents reference a specific model: `grep -r "model_id: claude-opus-5-5" ~/.config/opencode/agents/`

---

## Retiring an Agent

```bash
AGENT_ID="old-agent"
# 1. Archive it (don't delete immediately)
for TARGET in rabi@179.61.138.8 rabi@179.61.251.98; do
  ssh $TARGET "
    mv ~/.config/opencode/agents/$AGENT_ID \
       ~/.config/opencode/agents/_retired/$AGENT_ID-$(date +%Y%m%d)
  "
  echo "Archived on $TARGET"
done

# 2. After 30 days with no issues, remove the archive
# ssh $TARGET "rm -rf ~/.config/opencode/agents/_retired/$AGENT_ID-*"
```

---

## Failure Modes and Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Agent not loading after deploy | tools.json invalid JSON | `python3 -m json.tool tools.json` to find error |
| Agent loads but has no tools | tools.json `allowed_tools` empty | Add tools to the array |
| Cortex calls failing | Stale model_id in cortex.md | Check current catalog and update model_id |
| Agent behaviour wrong after update | Old .bak or cached version loaded | `rm agent.md.bak`; restart opencode session |
| Under 150 lines error | agent.md too short | Expand Workflow or Constraints sections |

---

## Related Skills

- `vault-index-maintainer` — parallel tool deployment pattern
- `rollback-executor` — restores previous agent.md from .bak on failure
- `cve-tracker` — checks if any agent's security guidance is stale
