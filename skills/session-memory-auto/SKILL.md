---
name: session-memory-auto
description: The claude-mem pattern for the fleet - hooks capture tool I/O, compress it into observations, auto-inject the most relevant prior context at SessionStart, and index the whole handoff+brain corpus with SQLite FTS5 so "what did we try for X" is one query instead of a file walk. Bias the store toward DECISIONS and rules, not transcript bytes. Removes reliance on the model remembering to persist (A05+B03 from the master upgrade table, P1). alwaysFire - fires on every session start (injection leg) and every substantive tool-use turn (capture leg), not on user request.
alwaysFire: true
---

# session-memory-auto - the mechanical session memory

You are running **session-memory-auto**. Your job is to make session memory
MECHANICAL: the capture of what happened, the compression of it into reusable
observations, the indexing of those observations, and the re-injection of the
relevant ones into the next session - none of it may depend on the model
"remembering to write things down."

**Why this skill exists (measured failures it closes):**

1. The fleet's Stop hook archives transcript BYTES, but nothing indexes them and
   nothing re-injects them. A "what did we try for the serve password two days
   ago" question is answered by walking files by hand - or by re-deriving the
   answer at full cost. The campaign measured stale in-context serve-password
   knowledge causing broken downstream consumers roughly 10 times (RGAP1).
2. The single most expensive observed failure class in the campaign log is a
   controller that lost its place and RE-DISPATCHED ALREADY-COMPLETED WORK.
   Every re-dispatch is a memory failure first.
3. Agent-discipline memory ("I'll append to the brain after this task") fires
   sometimes. Hooks fire ALWAYS. The strongest 2026 doctrine is: mechanical
   always-happens belongs in hooks, not in prompts or skills alone. The skill
   part of this file is the OPERATING PROCEDURE for the hooks; the hooks are
   the enforcement.

**What this skill is NOT:**

- NOT a memory vendor runtime. The master table (E04) killed Mem0/Zep/Letta
  per-turn LLM extraction: 1.5-4.2 s/turn cost and it scores LOWER than
  verbatim-plus-curation. This skill uses deterministic hooks + one cheap
  compression pass, never a per-turn LLM extraction call.
- NOT a transcript archive. Transcripts are bytes; observations are meaning.

## 0b) THE ONE-CARD OPERATING SUMMARY

```
CAPTURE  PostToolUse hook -> pending.jsonl (heads, redacted, fail-open)
COMPRESS Stop hook + threshold 25 -> observations (DECISION-biased, provenance-mandatory)
INDEX    hash-gated corpus crawl: handoffs + brain + beats -> same FTS5 table
INJECT   SessionStart recall (task terms + cwd) -> top-15 obs / 40 lines, capped
RECALL   smem recall "<terms>" --cwd .  = THE one query for prior art
VERIFY   V1 store+fts5 | V2 capture growth | V3 compression | V4 recall
         round-trip | V5 injection fires | V6 secret redaction
DEGRADE  no hooks -> S7 manual capture / S8 manual recall, reported honestly
NEVER    per-turn LLM capture | raw I/O in store | secrets in store |
         private per-seat stores | hunch-tier rows cited as evidence
```
  The store holds observations and pointers, not full tool output.
- NOT a replacement for the handoff system. context-handoff (the 11-section
  handoff file) is EPISODIC and human-readable. This store is SEMANTIC and
  decision-biased, and it complements handoffs; it does not replace them.

---

## 0) HARD RULES (survive compaction - never violate even if everything else is lost)

1. **Capture is a hook, never a habit.** If the only thing capturing tool I/O
   is the model's discipline, the capture does not exist. PostToolUse and Stop
   hooks must be wired in the runtime config before this skill is considered
   installed. A skill that describes capture without the hook is a wish.
2. **Compress to observations, never store raw tool I/O by default.** An
   observation is 1-6 sentences with a stable type tag (DECISION, RULE, GOTCHA,
   STATE, ARTIFACT, FAILURE, CREDENTIAL-REF) and provenance (session, tool,
   timestamp). Full blobs live in the transcript archive; the store holds
   meaning plus a pointer.
3. **Bias the store toward DECISIONS and RULES.** Episodic detail ("we then ran
   curl") is cheap to re-derive; the decision ("we picked PostgreSQL FTS5 over
   pgvector because the corpus is small and pure-SQLite deploys matter") is
   expensive to re-derive and is what gets lost. When a capture must be
   dropped, drop episodic first. A store with no DECISION rows is a defect.
4. **Injection is top-N RELEVANT, not "everything last session."** SessionStart
   injects at most N observations (default 8, hard cap 15), ranked by FTS5
   relevance against the current task prompt + cwd. A dump of yesterday's
   session is context pollution, not memory.
5. **Never store secrets in the observation store.** Keys, tokens, passwords,
   connection strings: store a CREDENTIAL-REF pointer (where the secret lives,
   never the secret). The store file is synced and read by many sessions.
6. **The store is append-only at capture time.** Observations are never
   silently edited or deleted; corrections are new observations with
   `supersedes: <rowid>` (the D08 fact-validity pattern). Pruning happens only
   in the consolidation pass (Section 8), which logs what it removed.
7. **Every claim about the past is queryable.** "Two days ago we tried X" must
   be answerable with one FTS5 query whose SQL you can paste. If the answer
   required a manual file walk, the index is stale - rebuild it (Section 6).
8. **FTS5 is the floor, not the ceiling.** A plain LIKE scan is the fallback
   when SQLite FTS5 is unavailable (Windows boxes without the extension);
   record which mode is active in the store header. Relevance ranking may
   degrade to recency+cwd-match, and the skill still counts as operational.
9. **Degraded mode is honest mode.** If hooks cannot run (no hook support in
   the runtime), the skill runs its MANUAL procedure (Section 7) and reports
   `capture: manual-fallback` in the gate line. Fake automatic capture that is
   actually nothing is the worst outcome.
10. **The store path is a run constant.** Default:
    `<workspace>/.session-memory/observations.db` (SQLite, WAL mode). Every
    skill that references memory points at the SAME file. Two stores for one
    fleet is the drift failure class again.

---

## 1) WHEN IT FIRES

Three legs, each with its own trigger:

| Leg | Trigger | Mechanism | Cost |
|-----|---------|-----------|------|
| Capture | EVERY substantive tool call (PostToolUse) | hook: append raw I/O pointer + queue line to `pending.jsonl` | ~1 ms, deterministic |
| Compress | Session Stop, and every ~25 queued items | hook (Stop) or threshold sweep: worker pass converts pending lines into observations | one pass, no LLM required for the deterministic subset |
| Inject | SessionStart of any session | hook: query FTS5 with task prompt + cwd, paste top-N into the session preamble | ~10 ms, deterministic |

Fires ALWAYS on all three (alwaysFire). There is no situational gating: the
whole point (A05 verdict, P1) is killing reliance on the model deciding when
memory matters.

Manual-fire (no hook runtime): the model runs `S7 MANUAL CAPTURE` at the end of
any substantive task and `S8 MANUAL INJECT` at the start of any task that might
have prior art. Report `capture: manual-fallback` whenever manual is in play.

---

## 2) ARCHITECTURE - the three legs in one diagram

```
   tool call (Bash/Read/Edit/WebFetch...)
        |
        v
  [PostToolUse hook] --append--> pending.jsonl   (leg 1: capture, bytes only)
        |
        v
  [Stop hook / threshold sweep]
        |  compress: pending.jsonl -> observations
        v
  observations.db (SQLite WAL)
     - observations table (typed, provenance-stamped)
     - observations_fts (FTS5 virtual table, indexed)
     - facts table (fact, as_of, source, superseded_by)   [D08 pattern]
        ^
        |  SELECT ... MATCH '<task terms>' ORDER BY rank LIMIT N
        |
  [SessionStart hook] --top-N--> next session preamble  (leg 3: inject)
        |
        v
  (same corpus indexed: docs/handoffs/*.md + brain knowledge notes
   are CRAWLED into the FTS index too - Section 6)
```

Division of labor with neighbors:

- `context-handoff` writes the 11-section handoff file. This skill indexes it
  and injects a pointer to it when relevant. Handoff = full prose for a human
  or fresh session to READ; observation = a queryable unit.
- The brain (`agency-brain` knowledge notes) holds durable, curated knowledge.
  This skill indexes those notes AND captures the raw decisions that the brain
  note may only get at the weekly review. Brain is curated; store is raw-ish.
- `session-monitor-auto` consumes the beats.md ledger; that ledger is also
  crawled into the index (its evidence rows are gold for "what side-effect
  proved X").

---

## 3) THE STORE - schema and rules

Single SQLite file, WAL mode, FTS5 if available. Full DDL in Appendix A. The
load-bearing tables:

```
observations(
  id INTEGER PRIMARY KEY,
  ts TEXT,                -- ISO 8601 UTC
  session_id TEXT,
  cwd TEXT,
  type TEXT,              -- DECISION|RULE|GOTCHA|STATE|ARTIFACT|FAILURE|CREDENTIAL-REF
  text TEXT,              -- 1-6 sentences, the compressed observation
  tool TEXT,              -- originating tool, e.g. "Bash"
  source_ptr TEXT,        -- pointer to the raw artifact (file:line or log path)
  confidence TEXT,        -- measured | directional | hunch
  supersedes INTEGER      -- id of the observation this corrects (NULL ok)
)

observations_fts USING fts5(text, cwd UNINDEXED, obs_type UNINDEXED)
  -- content table wired via triggers; INSERT/DELETE kept in sync
```

Type definitions - use these exactly, they are the retrieval vocabulary:

| Type | Stores | Example |
|------|--------|---------|
| DECISION | a choice + why + what was rejected | "Picked FTS5 over pgvector: single-file store beats a Postgres dep for fleet portability" |
| RULE | an operating rule learned by failure | "Never parse a fixed-length tail; anchor at the gates line" |
| GOTCHA | a trap + symptom + fix | "PS interpolates $? locally over ssh; single-quote the remote script" |
| STATE | durable run state worth recall | "Deploy 2026-11-13 shipped commit 9f2c1a; rollback = symlink to previous" |
| ARTIFACT | where a thing lives | "PANIC logo SVGs at /srv/dr/projects/panic-logos (557 files, md5-verified)" |
| FAILURE | a measured failure + cost | "$59/3h runaway: no budget gate was armed" |
| CREDENTIAL-REF | where a secret LIVES (never the secret) | "Soniox key in VPS ~/.env, key SONIOX_API_KEY" |

Compression rules for the worker pass:

- One observation per distinct fact. Never merge a decision and its evidence
  into one blob - the DECISION row cites the FAILURE row by pointer.
- Keep the operator's own words for the verdict part ("superseded_by pricing
  dead"), keep the mechanical context (file, command, date) neutral.
- If a pending line is pure noise (a `ls` with no outcome), drop it and log the
  drop count. Compression is curation, not archival.
- Every observation carries `cwd`. Relevance ranking weights cwd match heavily -
  the same project's history beats an unrelated box's history.

---

## 4) LEG 1 - CAPTURE (PostToolUse hook)

### 4.1 Hook config (OpenCode `opencode.json` shape; Claude Code equivalent in Appendix B)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash|Read|Edit|Write|WebFetch|Grep",
        "hooks": [
          { "type": "command",
            "command": "python3 %USERPROFILE%/.config/opencode/hooks/smem-capture.py" }
        ]
      }
    ]
  }
}
```

The hook receives the tool-call event on stdin (tool name, args, truncated
result) and appends ONE line to `pending.jsonl`:

```
{"ts":"2026-11-13T09:14:02Z","session":"ses_ab12","cwd":"/srv/dr/projects/x",
 "tool":"Bash","args_head":"git push origin main",
 "result_head":"To github.com:... 9f2c1a..3d77e0","rc":0}
```

Head-truncation rules: args_head 200 chars, result_head 400 chars. The hook
NEVER executes model code, NEVER calls the network, and NEVER blocks the tool
result (fail-open: a broken capture hook must not break the session - log to
`smem-errors.log` and exit 0).

### 4.2 What NOT to capture

- File READS of pure reference material (node_modules, vendor, .git objects).
  The matcher list excludes paths via a denylist in the hook config.
- Any tool result whose head matches a secret pattern (`sk-`, `AKIA`, `-----BEGIN`,
  `ghp_`, `Bearer `): the hook replaces the head with `[REDACTED-MATCH]`. The
  capture layer is the LAST line of secret defense, not the first.

### 4.3 Verification that capture works

The guard-does-not-exist-until-proven-to-fail rule applies. Prove the hook by
re-introducing traffic and watching the file grow:

```
# 1. note size
wc -l <workspace>/.session-memory/pending.jsonl
# 2. run any matched tool (e.g. Bash "echo smem-probe")
# 3. re-count - MUST be +1 within one tool turn
wc -l <workspace>/.session-memory/pending.jsonl
```

Two runs with no growth = the hook is not wired (check the runtime actually
loaded the config root you edited - MCP install is per config root, and so is
the hook config; verify per root, never per machine).

---

## 5) LEG 2 - COMPRESS (Stop hook + threshold sweep)

### 5.1 The worker pass

Triggered by the Stop hook when `pending.jsonl` has lines, OR by any session
when the pending count crosses 25 (threshold sweep - a crashed Stop hook must
not lose a long session's worth of capture).

The worker is a deterministic Python pass by default (no LLM):

- Lines whose result shows a completed state change (git push, file write,
  deploy probe 200, test suite exit code) become STATE/ARTIFACT observations.
- Lines whose args contain decision markers (`chose|picked|decided|because|
  instead of|rejected`) are flagged `needs-llm: true` and held in a small
  queue; at most ONE summarization call per session compresses that queue into
  DECISION observations. If no LLM is available, they are stored verbatim with
  type FAILURE-hint text and confidence `hunch` - never dropped.
- Secret-pattern lines are redacted before any LLM sees them (Rule 0.5).

### 5.2 Provenance is mandatory

Every observation stores `source_ptr` - the pending.jsonl line offset (which
itself carries session + ts). An observation without provenance is unverifiable
and is rejected by the compressor. This is what makes Rule 0.7 possible: the
query answers "what", the provenance answers "prove it".

### 5.3 Verification of compression

```
sqlite3 <store> "SELECT COUNT(*) FROM observations;"          # grew
sqlite3 <store> "SELECT COUNT(*) FROM observations WHERE type='DECISION';"  # > 0 for any real session
sqlite3 <store> "SELECT COUNT(*) FROM pending_lines_pending;" # shrank to 0 (or small tail)
```

A store that grows observations but never DECISION rows means the compressor is
storing bytes, not meaning - tune the decision markers (Section 5.1).

---

## 6) THE FTS INDEX - over the WHOLE corpus, not just the store

The master table's D07 is folded in here: the index covers
`docs/handoffs/*.md`, the brain knowledge notes (`web-dev/knowledge/`,
`seo/knowledge/`, `cybersec/knowledge/`, `llc/knowledge/` on the Obsidian
junction), and `*/beats.md` ledgers, in addition to `observations`. Crawl:

```
python3 %USERPROFILE%/.config/opencode/hooks/smem-index.py --corpus
```

- Corpus files are chunked (~1200 chars, 100-char overlap) into the same FTS5
  table with `kind='corpus'` and their path as provenance; store observations
  are `kind='obs'`.
- The crawl is incremental: hash each file; skip unchanged. A full rebuild is
  `--rebuild` and takes seconds at fleet scale (thousands of chunks).
- One search entry point - the `smem` query (below) - is THE recall path for
  "what did we try for X". This replaces the file-walk answer.

### 6.1 The one recall query

```
python3 %USERPROFILE%/.config/opencode/hooks/smem.py recall "<terms>" --cwd . --top 8
```

Order of results: FTS5 bm25 rank, boosted by (+3 cwd exact match, +2
kind='obs' DECISION over FAILURE over STATE, +1 within 14 days, -1 per 30 days
of age). The boost table lives in the script header and is versioned - do not
tune it per query.

---

## 7) LEG 3 - INJECT (SessionStart)

The SessionStart hook runs the recall query with the session's opening task
text (and cwd) and injects a block no longer than 40 lines:

```
[session-memory] top prior art for this task (N obs, corpus M):
1. [DECISION 2026-11-11] Picked FTS5 over pgvector ... (obs#412)
2. [GOTCHA 2026-11-09] pkill self-kill: bracket-trick the pattern ... (obs#388)
...
Full prose: docs/handoffs/LATEST.md | query: smem recall "<terms>"
```

Caps: 15 observations hard max, 40 injected lines hard max. If recall returns
nothing (new project, fresh box), inject nothing - an empty block is noise.

Anti-loop rule: injected observations carry a marker line so the capture hook
skips them (recording the injection of memory into the memory store is the
hall-of-mirrors bug).

---

## 8) MANUAL FALLBACK PROCEDURES (no hook runtime)

S7 MANUAL CAPTURE (run at the end of any substantive task):

1. Write 2-6 observations straight into the store:

```
sqlite3 <store> "INSERT INTO observations(ts,session,cwd,tool,source_ptr,
confidence,type,text) VALUES(datetime('now'),'manual','<cwd>','model',
'<task or handoff path>','directional','DECISION','<one fact, 1-2 sentences>');"
```

2. At least one DECISION for any real task. If you cannot name one decision,
   the task was likely trivial - fine, but say so.
3. Re-run the corpus index if a handoff/brain file changed.

S8 MANUAL RECALL (start of any task that might have prior art):

```
sqlite3 <store> "SELECT rowid,type,date(ts),text FROM observations_fts
 WHERE observations_fts MATCH '<terms>' ORDER BY rank LIMIT 8;"
```

If sqlite3 CLI is unavailable, Python `sqlite3` stdlib is the fallback
(everywhere has it). If BOTH fail, fall back to grep over `docs/handoffs/` and
say `recall: grep-fallback` honestly.

---

## 9) VERIFICATION COMMANDS (the installation gate)

An install is DONE only when all of these produce real output:

```
# V1. store exists, WAL on, FTS5 available
sqlite3 <store> "PRAGMA journal_mode; SELECT 'fts5-ok' FROM sqlite_master
 WHERE name='observations_fts';"
#    expect: wal  +  fts5-ok

# V2. capture leg fires (Section 4.3 growth test) - expect count +1

# V3. compression converts pending to observations - expect pending ~0

# V4. recall round-trip
smem recall "serve password" --top 3     # expect >=1 row after a real session
#    a fresh-box install may legitimately return 0 rows; run one tagged
#    session first, then re-run. 0 rows on a used box = index stale.

# V5. injection leg fires
#    open a NEW session and confirm the [session-memory] block appears
#    (or confirm honestly it returned nothing for a fresh cwd)

# V6. secret guard: run a tool that echoes a fake key, then
sqlite3 <store> "SELECT COUNT(*) FROM observations WHERE text LIKE '%sk-test%';"
#    expect 0 - the capture redacted it.
```

If V2 fails, the hooks are not loaded in the ACTIVE config root. The config
root law applies: apply to ALL roots on ALL boxes (dual-root-config-law),
verify per root, never per machine.

---

## 10) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| pending.jsonl grows, observations never grow | Stop hook missing or failing silently | wire Stop hook; check smem-errors.log; threshold sweep will recover pending lines |
| recall returns 0 on a box with months of history | corpus never indexed / db path mismatch | run --corpus; confirm the store path constant is the same one the hooks use |
| same observation duplicated dozens of times | capture hook fired on its own output; injection not marked | add injection-skip marker; dedup pass: `DELETE FROM observations WHERE rowid NOT IN (SELECT MIN(rowid) FROM observations GROUP BY type,text)` - log the count removed |
| old decision contradicts a new one and both surface | missing supersedes chain | new observation must set supersedes=<old id>; recall query must prefer rows with no successor |
| injection block eats half the context | N/lines caps not enforced in hook | enforce 15 obs / 40 lines hard caps in the SessionStart hook |
| secrets found in store text | capture redaction pattern list stale | extend pattern list; redact retroactively: UPDATE with a redaction sweep; treat the leaked key as ROTATED (secret-rotation skill), never just deleted |
| store locked / database is locked | two writers, no WAL | PRAGMA journal_mode=WAL once; hooks open with busy_timeout=2000 |
| Windows: hook python not found | PATH differs per config root | pin absolute python path in hook config; verify per root |

---

## 11) ANTI-PATTERNS

- PER-TURN LLM SUMMARIZATION as the capture mechanism. E04 measured it: 1.5-4.2
  s/turn and WORSE recall than verbatim+curate. Capture verbatim heads
  deterministically; compress in one batched pass.
- STORING FULL TOOL OUTPUT "in case." That is the transcript archive's job;
  the store holds pointers + compressed meaning.
- A PRIVATE store per seat. The store is fleet-level (per workspace); seats
  share it. Cross-seat "what did the recon seat find" queries are the point.
- RECALL WITHOUT cwd WEIGHT. Unanchored recall floods with other projects' noise.
- TRUSTING UNVERIFIED OBSERVATIONS. confidence is a first-class column; a
  `hunch` observation is a hypothesis, never a citation in a report.
- REPLACING THE BRAIN. The brain's curated knowledge notes stay authoritative
  for durable doctrine; observations are the fast layer. The consolidation
  pass (D09, nightly) promotes stable observations INTO brain notes - the
  store feeds the brain, never the reverse.

---

## 12) WORKED EXAMPLES

### 12.1 The serve-password incident, replayed under this skill

WITHOUT the skill: session learns the rotated opencode-serve password at 09:00;
a compacted continuation at 14:00 uses the stale in-context password; 10
downstream consumers break over the campaign.

WITH the skill: at 09:00 the capture leg stores

```
{type: STATE, text: "opencode serve restarted; password rotated; new pw in
/tmp/oc-serve-1140.log (validated: HTTP 200, len 24)", confidence: measured,
source_ptr: pending.jsonl#18342}
```

At 14:00 the SessionStart recall on task text "continue deploy verification"
ranks it first (cwd + recency + STATE boost). The injected block carries it in
one line. Zero downstream breakage, zero file walking.

### 12.2 "What did we try for the excerpt truncation bug?"

Query: `smem recall "excerpt truncation tail parse" --top 5`

Returns (real campaign rows): GOTCHA 2026-11-11 excerpt-tail-anchoring (3 false
negatives), RULE parse the FULL final text, DECISION anchor at the max-field
gates line within last ~1600 chars. The answer took one query, not three
sessions of re-derivation. This is the product of the skill in one sentence.

### 12.3 Manual fallback on a fresh Windows box (hooks not yet wired)

Session ends after a deploy fix. Operator (the model) runs the S7 INSERT for
one DECISION ("v14.2 plugin injects full 14-field gates line; drift D3 fix")
and one ARTIFACT ("plugin md5 5f98f693346aa4592..."). Reports
`capture: manual-fallback` in the gates line. Honest degraded capture beats
silently missing capture.

---

## 13) GATES-LINE CONTRACT

This skill's slot in the canonical 14-field line (evidence-in/gate-format-spec.md):

- Field 1 `godmode:activation` stays the chain's; this skill never owns it.
- This skill contributes to field 14 `brain:` when it wrote observations/brain
  notes, and to the dims when its recall fed the answer:
  - `dim-completeness` may cite recall coverage ("prior art checked: 5 obs").
  - A done-claim that IGNORED a contradicting stored observation is a
    completeness defect - auto-check: recall the task terms before any
    done-claim; a stored FAILURE contradicting the claim must be addressed in
    the answer or the claim is dishonest.
- The skill never emits its own gate line; it feeds the parent's. In the
  per-task gate line format, this skill's slot reads:
  `memory: captured N obs, injected M (or skipped: why)` - appended ONLY when
  the run carries this skill in its chain; otherwise omitted entirely.

When reporting "skipped", the honest whys are:
`skipped: no hook runtime` (manual fallback used - say so),
`skipped: trivial ask` (no substantive tool I/O this turn),
`skipped: store unwritable` (report the path that failed).

---

## 14) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| context-handoff | handoff files are corpus input; injection points at LATEST.md |
| session-monitor-auto | beats.md is corpus input; monitor stalls cite the ledger |
| goal | goal-loop iterations read recall before designing iteration N+1 (kill repeated ideas) |
| honestscore | "NOT verified" items get stronger: past observations with confidence=measured are evidence; hunch-tier are not |
| secret-rotation | CREDENTIAL-REF rows point at vault/env locations; rotation updates the REF, never the secret |
| ideas | Step 0 preflight (mistake-memory grep) is now one FTS query instead of a brain walk |
| subagents | spawn briefs inject recalled prior art via lesson-recall; children write observations to the shared store with their seat id in session |

---

## 15) OPEN ITEMS (honest limits)

- Relevance boost table is heuristic; no head-to-head benchmark of fleet-scale
  FTS recall vs plain RAG exists in the campaign log (runlog-DR7 caveat stands).
  Verdict: FTS5 because the corpus is files we own and the dep must stay
  zero-service; revisit if recall precision measurably disappoints.
- Windows hook plumbing is per config root and per python install; the
  absolute-path pin is a workaround, not a fix.
- Consolidation/dreaming (D09) is a scheduled companion skill, not built here;
  until it exists, the store grows append-only and the FTS index absorbs the
  size fine (SQLite handles millions of rows; context injection is capped).

---

## APPENDIX A - FULL STORE SCHEMA (copy-paste install)

```sql
-- file: <workspace>/.session-memory/observations.db
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 2000;

CREATE TABLE IF NOT EXISTS observations (
  id          INTEGER PRIMARY KEY,
  ts          TEXT NOT NULL,            -- ISO 8601 UTC
  session_id  TEXT NOT NULL DEFAULT 'manual',
  cwd         TEXT NOT NULL DEFAULT '',
  tool        TEXT NOT NULL DEFAULT 'model',
  source_ptr  TEXT NOT NULL,            -- provenance: pending.jsonl#N, file path, or task ref
  confidence  TEXT NOT NULL DEFAULT 'hunch'
              CHECK (confidence IN ('measured','directional','hunch')),
  type        TEXT NOT NULL
              CHECK (type IN ('DECISION','RULE','GOTCHA','STATE','ARTIFACT',
                              'FAILURE','CREDENTIAL-REF')),
  text        TEXT NOT NULL,
  supersedes  INTEGER REFERENCES observations(id)
);
CREATE INDEX IF NOT EXISTS idx_obs_ts    ON observations(ts);
CREATE INDEX IF NOT EXISTS idx_obs_type  ON observations(type);
CREATE INDEX IF NOT EXISTS idx_obs_cwd   ON observations(cwd);
CREATE INDEX IF NOT EXISTS idx_obs_sup   ON observations(supersedes);

CREATE TABLE IF NOT EXISTS pending_lines (
  line_id INTEGER PRIMARY KEY,          -- offset in pending.jsonl at ingest
  raw     TEXT NOT NULL,
  flags   TEXT NOT NULL DEFAULT '',     -- e.g. 'needs-llm'
  ingested_at TEXT
);

CREATE VIRTUAL TABLE IF NOT EXISTS observations_fts USING fts5(
  text, cwd UNINDEXED, obs_type UNINDEXED, kind UNINDEXED, ptr UNINDEXED
);
-- corpus chunks live in the same FTS table with kind='corpus' and the file
-- path in ptr; store observations use kind='obs' and 'obs#<id>' in ptr.

CREATE TRIGGER IF NOT EXISTS obs_ai AFTER INSERT ON observations BEGIN
  INSERT INTO observations_fts(text,cwd,obs_type,kind,ptr)
  VALUES(new.text,new.cwd,new.type,'obs','obs#'||new.id);
END;

CREATE TABLE IF NOT EXISTS meta (
  k TEXT PRIMARY KEY, v TEXT NOT NULL
);
INSERT OR REPLACE INTO meta VALUES('schema_version','1');
INSERT OR REPLACE INTO meta VALUES('fts_mode','fts5');   -- or 'grep-fallback'
```

Query examples that must work verbatim after install:

```sql
-- recall (bm25-ranked, cwd-boosted)
SELECT rowid, obs_type, ts, text
  FROM observations_fts
 WHERE observations_fts MATCH ?
   AND (kind = 'obs' OR 1)
 ORDER BY rank, ts DESC
 LIMIT 8;

-- current-value view (respects supersedes chains)
SELECT o.* FROM observations o
 WHERE NOT EXISTS (SELECT 1 FROM observations s WHERE s.supersedes = o.id)
 ORDER BY o.ts DESC;

-- dedup sweep (log the count BEFORE deleting)
SELECT COUNT(*) FROM observations
 WHERE rowid NOT IN (SELECT MIN(rowid) FROM observations GROUP BY type, text);
```

---

## APPENDIX B - HOOK WIRING PER RUNTIME

### B.1 OpenCode (opencode.json / config root)

```json
{
  "experimental": { "hooks": true },
  "hook": {
    "PostToolUse": [{ "command": "python3 <abs>/smem-capture.py" }],
    "Stop":         [{ "command": "python3 <abs>/smem-compress.py" }],
    "SessionStart": [{ "command": "python3 <abs>/smem-inject.py" }]
  }
}
```

The `python3` token MUST be an absolute interpreter path on Windows boxes
(`C:/Python314/python.exe` or the uv-managed shim). Relative PATH resolution
differs per config root - the dual-root law applies: apply to ALL roots,
verify per root.

### B.2 Claude Code (settings.json)

```json
{
  "hooks": {
    "PostToolUse": [{ "matcher": "Bash|Edit|Write",
      "hooks": [{ "type": "command", "command": "python3 <abs>/smem-capture.py" }] }],
    "Stop":        [{ "hooks": [{ "type": "command",
                      "command": "python3 <abs>/smem-compress.py" }] }],
    "SessionStart": [{ "hooks": [{ "type": "command",
                      "command": "python3 <abs>/smem-inject.py" }] }]
  }
}
```

Claude Code passes a JSON event object on stdin; the capture script reads
`tool_name`, `tool_input`, `tool_response` and writes the pending line. The
same script serves both runtimes because it only reads stdin JSON generically.

### B.3 Env contract for all hook scripts

| Env var | Meaning | Default |
|---------|---------|---------|
| SMEM_DB | store path override | `<workspace>/.session-memory/observations.db` |
| SMEM_PENDING | pending file | `<workspace>/.session-memory/pending.jsonl` |
| SMEM_MAX_INJECT_OBS | injection cap | 15 |
| SMEM_MAX_INJECT_LINES | injection line cap | 40 |
| SMEM_SWEEP_AT | threshold sweep size | 25 |

All hooks: fail-open (exit 0 on internal error, error to smem-errors.log).
A broken memory hook must never break a working session.

---

## APPENDIX C - CAPTURE SCRIPT SPECIFICATION (smem-capture.py contract)

Deterministic, stdlib-only, ~120 lines. Pseudocode of the contract (any
implementation must satisfy it):

```
read stdin JSON (tool name, tool_input, tool_output, session_id, cwd)
if tool in {Read} and path matches denylist (node_modules, .git, vendor,
  site-packages, .venv, __pycache__): exit 0
args_head  <- first 200 chars of json(tool_input)
result_head<- first 400 chars of str(tool_output)
for pat in SECRET_PATTERNS:  # sk-, ghp_, AKIA, Bearer , BEGIN ... PRIVATE KEY,
                             # password=, Authorization:
    args_head <- pat.sub("[REDACTED-MATCH]", args_head)   (same for result_head)
append json line to pending.jsonl with flock/lock retry x3 (busy_timeout)
if pending count >= SMEM_SWEEP_AT (25): exit code 42 (runtime may trigger the
  compress pass out-of-band; the Stop hook sweeps regardless)
exit 0 ALWAYS (fail-open)
```

---

## APPENDIX D - MORE TROUBLESHOOTING ROWS

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Injection block appears twice per session | SessionStart registered in two config roots | keep one root authoritative; dedup by marker line check in injector |
| recall latency > 200 ms at scale | index rebuilt on every query | query only; rebuild on corpus change (hash-gated) |
| DECISION text reads like a task log, not a decision | compressor prompt too generic | enforce the shape: "X over Y because Z; rejected W" - reject rows failing the shape to FAILURE-hint |
| Handoff corpus chunks drown out observations | boost table weighting | kind='obs' boost already +2; if still drowned, raise to +4 and log the change in meta table |
| Box cloned with .session-memory excluded | backup excludes dotdirs | add the dir to the backup include list (backup-routine); stores are small (MBs) |
| supersedes chain cycles (A supersedes B supersedes A) | manual double-insert race | cycle check in compressor: reject supersedes pointing at a row that already supersedes the new row's root |
| grep-fallback mode after disk full | FTS insert failed silently | meta fts_mode flips; alert in injection block header until rebuilt |

---

## APPENDIX E - WORKED EXAMPLE: FLEET ROLLOUT SEQUENCE

1. Author the three hook scripts; unit-test the capture script with a canned
   stdin event (expect exactly one pending line, redaction applied).
2. Install on ONE box, ONE config root. Run V1-V6. Paste outputs into the
   runlog. Nothing ships fleet-wide without V2's growth test passing twice.
3. Dual-root + second box: apply per root (3 roots x 2 boxes), verify per root
   (the campaign's drift law - JSONC round-trip parse + law-grep per root).
4. Seed pass: run `--corpus` over docs/handoffs + brain. Confirm recall
   round-trips on three known historical questions (serve password, excerpt
   truncation, budget runaway).
5. Add the skill's `memory:` slot to the chain template; go live. Watch one
   full session for hall-of-mirrors (injection capture) and dedup growth.

Rollback: remove the hooks from the config root(s); the store file is plain
SQLite and remains queryable. No data loss, no migration.

---

## APPENDIX F - CONFIDENCE TIERS AND WHAT THEY MAY SUPPORT

The `confidence` column is only useful if every consumer applies the SAME
rules. These are the rules:

| Tier | Set by | May be cited as | May NOT be cited as |
|------|--------|-----------------|---------------------|
| measured | a pasted tool result / exit code / count sits behind it (source_ptr resolvable) | evidence in a report; feeds honestscore caps | - |
| directional | a decision or pattern with stated reasoning, no fresh tool result | a recommendation with the trade-off named | proof; a number |
| hunch | flagged needs-llm stored verbatim; or recall output with no source_ptr | a hypothesis to test | anything |

Enforcement points:

1. The compressor sets the tier; the injector prints it; a report that cites a
   hunch-tier row as fact is caught in review the same way an unsourced CVE
   claim is caught.
2. honestscore's evidence tiers map 1:1: measured -> no cap, directional ->
   <=7.0 for the dimension it supports, hunch -> reasoning-only (<=6.0).
3. When two observations of different tiers conflict, the measured one wins;
   if both are hunch-tier, BOTH are surfaced as open questions, never
   silently one-picked.

## APPENDIX G - THE INJECTION TEMPLATE (exact shape)

The SessionStart injector emits EXACTLY this block shape, so downstream
consumers (lesson-recall, subagents briefs) can parse it:

```
[session-memory] prior art for "<task-terms>" (cwd=<cwd>)
obs:
  1. [DECISION 2026-11-11 conf=measured] <text> (obs#412)
  2. [GOTCHA 2026-11-09 conf=measured] <text> (obs#388)
corpus:
  1. docs/handoffs/2026-11-11-excerpt-truncation.md (chunk 4): <one-line>
limits: N obs shown of K matches; full corpus: smem recall "<terms>"
inject-marker: smem-inject-v1
```

Parsing rules for consumers:

- The `inject-marker` line is how the capture hook skips injections (no
  hall-of-mirrors) and how a duplicate-injection bug is detected.
- `obs#` ids are stable rowids; citing `(obs#412)` in a report makes the claim
  auditable with one query.
- The `limits:` line must state K (total matches) even when N < K, so a
  session that needs more context knows the recall was truncated, not empty.

## APPENDIX H - CROSS-SEAT SHARE PROTOCOL (subagents integration detail)

When a crew runs (subagents skill, floor 5 / ceiling 30), the shared store is
the inter-seat memory beyond the spawn briefs:

1. Every seat writes observations tagged with its seat id
   (`session_id = '<parent-ses>#<seat-Sn>'`). The seat id rides the brief
   (subagents skill puts it in the mandate header).
2. The orchestrator's merge step (subagents S7) runs a recall restricted to
   the run's seats before final assembly:
   `smem recall "<task terms>" --seat-scope '<run-slug>' --top 20` - catches
   decisions a seat made but never surfaced in its final JSON.
3. A seat that discovers a DECISION-level fact mid-task writes it to the store
   IMMEDIATELY (one INSERT), not only in its final JSON - the JSON is for
   status, the store is for meaning. This is the difference between a merge
   that uses a finding and one that rediscovers it.
4. Wave N+1 briefs get the run's own observations injected by lesson-recall
   (see that skill) - the floor crew pays zero extra tokens for this because
   the injection is capped at the same 15-obs limit.

Failure class this kills (measured): the cross-seat "three agents ran the same
search" failure - the subagents skill handles it by sibling OUT-lists in the
brief; this skill handles the residual case where the same search is repeated
ACROSS waves by a fresh session that never saw wave 1.
