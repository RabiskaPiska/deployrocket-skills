---
name: session-bootstrapper
description: New-VPS or new-machine first-message bootstrap flow. Trigger on: "bootstrap this machine", "set up the fleet on this box", "install the fleet", first message on a freshly provisioned VPS, any machine where skill count or agent count is 0. Covers OS detection, pull-sync steps, tool vault bootstrap, and verification checks (agent count, skill count, tool count).
---

# Session Bootstrapper

## Trigger

Fire this skill when ANY of these are true:
- The session is the first on a newly provisioned VPS or machine
- A machine reports 0 skills, 0 agents, or missing vault tools
- The user says "bootstrap", "set up the fleet", "install skills", "fresh machine"
- A skill invocation fails because the skill file is not found
- Pull-sync is returning exit code 2 or "agents/index.json not found"

Never skip this skill just because you "think the box is already set up" — verify the counts first.

## Instructions

### Phase 1 — OS Detection

Determine the shell environment BEFORE running any setup command:

```bash
# Run this first — works on both bash and PowerShell
uname -s 2>/dev/null && echo "UNIX" || echo "WINDOWS"
```

| Result | Environment | Config root |
|--------|-------------|-------------|
| Linux/Darwin | bash/zsh | `~/.config/opencode/` |
| WINDOWS | PowerShell | `%APPDATA%\opencode\` or `C:\Users\<user>\.config\opencode\` |

For Linux VPS (most common case), all commands below use bash. For Windows (this PC), use PowerShell equivalents noted in brackets.

### Phase 2 — Verify Current State

Before installing anything, check what already exists:

```bash
# Linux — check existing skill, agent, and tool counts
SKILL_COUNT=$(find ~/.config/opencode/skills/ -name "SKILL.md" 2>/dev/null | wc -l)
AGENT_COUNT=$(find ~/.config/opencode/agents/ -name "*.md" 2>/dev/null | wc -l)
echo "Skills: $SKILL_COUNT | Agents: $AGENT_COUNT"

# Check opencode version
opencode --version 2>/dev/null || echo "opencode not in PATH"

# Check if pull-sync script exists
ls ~/.config/opencode/pull-sync.sh 2>/dev/null && echo "pull-sync: EXISTS" || echo "pull-sync: MISSING"
```

Expected healthy state for a fleet VPS:
- Skills: ≥25
- Agents: ≥5
- opencode: any version (confirm it responds)

If healthy, stop here — no bootstrap needed, just verify the specific missing component.

### Phase 3 — Install OpenCode (if missing)

```bash
# Linux — install opencode via npm (Node 18+ required)
node --version || (curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt-get install -y nodejs)
npm install -g opencode-ai@latest
opencode --version

# Confirm config directory exists
mkdir -p ~/.config/opencode/skills
mkdir -p ~/.config/opencode/agents
mkdir -p ~/.config/opencode/memory
mkdir -p ~/.config/opencode/rules
```

### Phase 4 — Bootstrap the Tool Vault

The fleet uses a central brain repo on GitHub. Pull it:

```bash
# Pull the deployrocket central brain (contains skills, agents, tools)
cd /srv/dr
git clone https://github.com/RabiskaPiska/deployrocket-central-brain.git brains/deployrocket-central-brain 2>/dev/null \
  || (cd /srv/dr/brains/deployrocket-central-brain && git pull --rebase)

# Verify brain pulled correctly
ls /srv/dr/brains/deployrocket-central-brain/ | head -10
```

### Phase 5 — Install Skills via Pull-Sync

```bash
# Check if pull-sync script exists in the brain
ls /srv/dr/brains/deployrocket-central-brain/scripts/pull-sync.sh 2>/dev/null \
  && echo "FOUND" || echo "MISSING — create manually"

# If pull-sync exists, run it
bash /srv/dr/brains/deployrocket-central-brain/scripts/pull-sync.sh

# Verify skill count after sync
find ~/.config/opencode/skills/ -name "SKILL.md" | wc -l
```

**If pull-sync is missing or broken (cold-path fix):**

The known cold-path bug: pull-sync reads `index.json` which returns path `"agents/index.json"` causing a 404 and exit code 2. Fix:

```bash
# Manually copy skills from brain to config
SRC="/srv/dr/brains/deployrocket-central-brain/skills"
DST="$HOME/.config/opencode/skills"

if [ -d "$SRC" ]; then
  rsync -av "$SRC/" "$DST/"
  echo "Copied $(find $DST -name SKILL.md | wc -l) skill files"
else
  echo "ERROR: skills source not found at $SRC"
fi
```

### Phase 6 — Install Agents

```bash
# Copy agent definitions from brain
SRC="/srv/dr/brains/deployrocket-central-brain/agents"
DST="$HOME/.config/opencode/agents"

if [ -d "$SRC" ]; then
  rsync -av "$SRC/" "$DST/"
  echo "Copied $(ls $DST/*.md 2>/dev/null | wc -l) agent files"
else
  echo "No agents directory in brain — check brain structure"
fi
```

### Phase 7 — Install AGENTS.md (Global Instructions)

```bash
# The AGENTS.md must exist at the config root
SRC="/srv/dr/brains/deployrocket-central-brain/config/AGENTS.md"
DST="$HOME/.config/opencode/AGENTS.md"

if [ -f "$SRC" ]; then
  cp "$SRC" "$DST"
  echo "AGENTS.md installed: $(wc -l < $DST) lines"
else
  echo "WARNING: No AGENTS.md in brain — fleet global instructions missing"
fi
```

### Phase 8 — Bootstrap the Launch Script

```bash
# Ensure the ultracode launcher exists
if [ ! -f /srv/dr/launch-ultra.sh ]; then
  cat > /srv/dr/launch-ultra.sh << 'EOF'
#!/bin/bash
# launch-ultra.sh — start an ultracode session on this VPS
cd /srv/dr
opencode --effort xhigh --model claude-opus-5-5 "$@"
EOF
  chmod +x /srv/dr/launch-ultra.sh
  echo "launch-ultra.sh created"
else
  echo "launch-ultra.sh already exists"
fi
```

### Phase 9 — Verification Checks

Run ALL of these. A bootstrap is not done until every check passes:

```bash
echo "=== BOOTSTRAP VERIFICATION ==="

# 1. OpenCode responds
opencode --version && echo "[PASS] opencode" || echo "[FAIL] opencode not found"

# 2. Skill count
SC=$(find ~/.config/opencode/skills/ -name "SKILL.md" 2>/dev/null | wc -l)
[ "$SC" -ge 25 ] && echo "[PASS] skills: $SC" || echo "[WARN] skills: $SC (expected >=25)"

# 3. Agent count
AC=$(ls ~/.config/opencode/agents/*.md 2>/dev/null | wc -l)
[ "$AC" -ge 5 ] && echo "[PASS] agents: $AC" || echo "[WARN] agents: $AC (expected >=5)"

# 4. AGENTS.md exists
[ -f ~/.config/opencode/AGENTS.md ] && echo "[PASS] AGENTS.md" || echo "[FAIL] AGENTS.md missing"

# 5. Brain repo is clean (no uncommitted changes from a previous failed sync)
cd /srv/dr/brains/deployrocket-central-brain && git status --short | wc -l | \
  xargs -I{} bash -c '[ {} -eq 0 ] && echo "[PASS] brain repo clean" || echo "[WARN] brain repo has {} uncommitted files"'

# 6. Launch script executable
[ -x /srv/dr/launch-ultra.sh ] && echo "[PASS] launch-ultra.sh" || echo "[FAIL] launch-ultra.sh not executable"

echo "=== END VERIFICATION ==="
```

**Minimum passing bar:**
- opencode: PASS
- skills: ≥25 (WARN at <25, FAIL at 0)
- agents: ≥5 (WARN at <5)
- AGENTS.md: PASS
- launch-ultra.sh: PASS

### Phase 10 — Write Bootstrap Record

After a successful bootstrap, write a record so the next session knows this machine is set up:

```bash
cat > ~/.config/opencode/bootstrap-record.txt << EOF
Bootstrap completed: $(date -I)
Host: $(hostname)
Skills: $(find ~/.config/opencode/skills/ -name "SKILL.md" | wc -l)
Agents: $(ls ~/.config/opencode/agents/*.md 2>/dev/null | wc -l)
OpenCode: $(opencode --version 2>/dev/null)
Brain: $(cd /srv/dr/brains/deployrocket-central-brain && git log -1 --format="%h %s" 2>/dev/null || echo "unknown")
EOF
cat ~/.config/opencode/bootstrap-record.txt
```

### Windows Bootstrap (this PC — PowerShell)

For the main Windows PC at `C:\Users\rabim`:

```powershell
# Check current state
$skillCount = (Get-ChildItem -Recurse "C:\Users\rabim\.config\opencode\skills" -Filter "SKILL.md" -ErrorAction SilentlyContinue).Count
$agentCount = (Get-ChildItem "C:\Users\rabim\.config\opencode\agents" -Filter "*.md" -ErrorAction SilentlyContinue).Count
Write-Host "Skills: $skillCount | Agents: $agentCount"

# Copy new skills authored on this session
$newSkillsDir = "C:\Users\rabim\OneDrive\Desktop\NEW RESULTS\04-FLEET-COMPLETE\skills-authored"
Get-ChildItem $newSkillsDir -Directory | ForEach-Object {
    $dest = "C:\Users\rabim\.config\opencode\skills\$($_.Name)"
    New-Item -ItemType Directory -Force -Path $dest | Out-Null
    Copy-Item "$($_.FullName)\SKILL.md" "$dest\SKILL.md" -Force
    Write-Host "Installed: $($_.Name)"
}
```

## References

- Related skills: `cold-pull-fix` (pull-sync path bug), `context-curator` (load context after bootstrap), `fleet-monitor` (heartbeat after bootstrap)
- VPS fleet: web-dev `179.61.138.8`, knowledge `179.61.251.98`
- Brain repo: `RabiskaPiska/deployrocket-central-brain`
- Launch script: `/srv/dr/launch-ultra.sh`
- Brain sync script: `/srv/dr/brain-sync.sh`
- OpenCode docs: `https://claude.com/docs/` (for latest install instructions)

## Scripts

```bash
#!/bin/bash
# full_bootstrap.sh — idempotent bootstrap for a Linux VPS fleet node
# Safe to run multiple times; skips steps that are already done
set -e

echo "[1/6] Checking opencode..."
command -v opencode || npm install -g opencode-ai@latest

echo "[2/6] Ensuring config dirs..."
mkdir -p ~/.config/opencode/{skills,agents,memory,rules}

echo "[3/6] Pulling brain repo..."
mkdir -p /srv/dr/brains
cd /srv/dr/brains
git clone https://github.com/RabiskaPiska/deployrocket-central-brain.git 2>/dev/null \
  || (cd deployrocket-central-brain && git pull --rebase)

echo "[4/6] Syncing skills..."
rsync -a /srv/dr/brains/deployrocket-central-brain/skills/ ~/.config/opencode/skills/

echo "[5/6] Syncing agents..."
rsync -a /srv/dr/brains/deployrocket-central-brain/agents/ ~/.config/opencode/agents/ 2>/dev/null || true

echo "[6/6] Verification..."
SC=$(find ~/.config/opencode/skills/ -name "SKILL.md" | wc -l)
AC=$(ls ~/.config/opencode/agents/*.md 2>/dev/null | wc -l)
echo "DONE — skills=$SC agents=$AC host=$(hostname) date=$(date -I)"
```