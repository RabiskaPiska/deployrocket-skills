---
name: knowledge-agent
description: RAG query interface for the ChromaDB pentest knowledge base — 7 cloned sources covering payloads, WAF bypass, SQLi tamper scripts, bug bounty playbooks, and CVE tables. Query using natural language; returns ranked passages with source citations and confidence scores. Use when you need authoritative technique references, CVE details, tamper chains, or bug bounty methodology from the local corpus.
triggers:
  - query knowledge base
  - search CVE database
  - what does the KB say about
  - look up in knowledge base
  - search pentest knowledge base
  - KB query
  - knowledge base lookup
  - find in knowledge base
  - query local corpus
---

# KNOWLEDGE AGENT — RAG Query Interface

## Purpose

The knowledge agent is a local Retrieval-Augmented Generation (RAG) layer built on ChromaDB. It sits on top of the pentest knowledge base assembled by the fleet-complete run and answers natural-language queries against the full corpus of tool documentation, CVE tables, WAF bypass techniques, SQLi tamper chains, and bug bounty playbooks — without re-reading raw files.

**Latency (measured live 2026-10-30):** 532–727 ms per query.  
**Index state (as of 2026-10-31):** Collections initialized; initial ingestion of PayloadsAllTheThings and SecLists recommended before first query.

---

## Indexed Sources (7 cloned repositories)

| # | Source | Type | License | Path |
|---|--------|------|---------|------|
| 1 | swisskyrepo/PayloadsAllTheThings | Payload/bypass reference | MIT | `02-AUTOMATED-SOURCES/PayloadsAllTheThings/` |
| 2 | EnableSecurity/EvilWAF | WAF evasion payloads | Apache-2.0 | `02-AUTOMATED-SOURCES/EvilWAF/` |
| 3 | nicowillis/WAFNinja | WAF bypass automation | No license | `02-AUTOMATED-SOURCES/WAFNinja/` |
| 4 | sqlmapproject/sqlmap-tamper-collection | SQLi tamper scripts | GPLv2 | `02-AUTOMATED-SOURCES/sqlmap-tamper-collection/` |
| 5 | ngalongc/bug-bounty-reference | Bug bounty playbook | MIT | `02-AUTOMATED-SOURCES/Bug-Bounty-Playbook/` |
| 6 | BBLabs Academy | Bug bounty techniques | MIT | `02-AUTOMATED-SOURCES/BBLabs-Academy/` |
| 7 | Intigriti blog | Bug bounty CVE writeups | Public | `02-AUTOMATED-SOURCES/Intigriti-Blog/` |

**Priority weighting during retrieval:**
1. MASTER-REPORT.md — highest weight (synthesized, verified facts)
2. Per-tool technique files (TOOL-05 through TOOL-10) — high weight
3. PayloadsAllTheThings chunks — medium weight (raw payloads)
4. CVE table rows — medium weight (date-stamped)
5. BBLabs/Intigriti writeups — lower weight (narrative)

**Not indexed (HackerOne — F-20 closure):** HackerOne integration is skipped. The API returns HTTP 401 due to a base64 token decode bug combined with the API being restricted to personal profile accounts. The fix recipe is documented in F-20 but not applied — HackerOne is not on the critical path. See F-20 section below.

---

## Query Interface — rag_client.py

**Location:** `C:\Users\rabim\OneDrive\Desktop\NEW RESULTS\03-KNOWLEDGE-BASE\03-KNOWLEDGE-AGENT\rag_client.py`  
**VPS location:** `/srv/dr/projects/pentest-kb/rag_client.py` (after deployment)

### Commands

```bash
# Query the index (primary use)
python rag_client.py --query "how to bypass cloudflare WAF with sqlmap"

# Rebuild the index from source files
python rag_client.py --rebuild

# Check index health (document count, last build time)
python rag_client.py --status

# Add a single file without full rebuild
python rag_client.py --add-file path/to/new-technique.md
```

### Output Format (query mode)

```json
{
  "query": "how to bypass cloudflare WAF with sqlmap",
  "results": [
    {
      "score": 0.91,
      "source": "MASTER-REPORT.md#sqlmap-waf-bypass",
      "text": "Cloudflare WAF bypass with sqlmap: use --tamper=space2comment,versionedkeywords,randomcase,charunicodeencode with --random-agent"
    },
    {
      "score": 0.78,
      "source": "PayloadsAllTheThings/SQL Injection/WAF-Bypass.md",
      "text": "..."
    }
  ]
}
```

**Score interpretation:**
- ≥ 0.90 — high confidence match; use directly
- 0.70–0.89 — good match; verify against source
- 0.50–0.69 — weak match; treat as a hint only
- < 0.50 — no useful match; rebuild index or rephrase query

---

## Query Patterns (Proven Effective)

```bash
# CVE lookup
python rag_client.py --query "CVE-2026-19949 second order SQL injection WordPress"

# Tamper chain lookup
python rag_client.py --query "sqlmap tamper scripts for Cloudflare WAF"

# Technique lookup
python rag_client.py --query "ffuf vhost discovery Host header fuzzing"

# Exploit chain lookup
python rag_client.py --query "Metasploit Meterpreter reverse TCP payload"

# Bug bounty methodology
python rag_client.py --query "IDOR horizontal privilege escalation methodology"

# WAF bypass technique
python rag_client.py --query "ModSecurity bypass chunked encoding"
```

---

## Architecture

```
User query (natural language)
    │
    ▼
rag_client.py
    │  1. Embed query → all-MiniLM-L6-v2 (79.3 MB ONNX, cached)
    │  2. Retrieve top-K chunks → ChromaDB 1.5.9
    │     Collection: pentest-kb-2026
    │     Index: 02-AUTOMATED-SOURCES/rag-index/chroma.sqlite3
    │  3. Return ranked passages + source citations
    ▼
Caller (OpenCode agent, VPS session, or CLI)
```

**ChromaDB details:**
- Version: 1.5.9 (installed and verified)
- Collection name: `pentest-kb-2026`
- Index path: `03-KNOWLEDGE-BASE\02-AUTOMATED-SOURCES\rag-index\`
- SQLite file: `chroma.sqlite3` (present, collections initialized)
- Embedding model: all-MiniLM-L6-v2 (ONNX, downloaded to local cache)
- Chunk size: 512 tokens, overlap 64 tokens

---

## Dependencies and Installation

```bash
pip install chromadb>=0.5.0 sentence-transformers>=3.0.0 langchain-text-splitters>=0.2.0

# VPS deployment
pip3 install chromadb sentence-transformers langchain-text-splitters
mkdir -p /srv/dr/projects/pentest-kb/rag-index
```

**Note:** First build downloads ~90 MB model (all-MiniLM-L6-v2). Subsequent builds use the cached model. Plan for 2–5 min on first run on VPS. No API keys required — all models run locally.

---

## Index Rebuild

Run after any source update:

```bash
cd "C:\Users\rabim\OneDrive\Desktop\NEW RESULTS\03-KNOWLEDGE-BASE\03-KNOWLEDGE-AGENT"
python rag_client.py --rebuild
```

Expected output:
```
[*] Clearing existing index...
[*] Loading source files...
[*] Chunking documents (chunk_size=512, overlap=64)...
[*] Embedding and indexing N chunks...
[*] Done. Index written to ..\02-AUTOMATED-SOURCES\rag-index\
[*] Collection: pentest-kb-2026 | Documents: NNNN
```

---

## Quality Gate (3 mandatory verification queries)

Before claiming the RAG index is working:

```bash
# Must return CVE-2026-19949 section
python rag_client.py --query "sqlmap second order injection"

# Must return Host header fuzzing technique
python rag_client.py --query "ffuf vhost discovery"

# Must return payload/Meterpreter section
python rag_client.py --query "metasploit meterpreter reverse tcp"
```

All three must score ≥ 0.5. If any fail — rebuild the index.

---

## F-20: HackerOne Skip Documentation

**Issue:** HackerOne API returns HTTP 401 on all authenticated requests.

**Root cause (two factors):**
1. The API token must be decoded from base64 to raw bytes before use — the client was passing the encoded form, not the decoded token.
2. The HackerOne API for program searches is restricted to accounts associated with a personal security profile. The registered account is an organizational/API-only account and does not have a profile.

**Status:** SKIPPED — not on critical path. HackerOne data is lower priority than HackTricks, GitHub GHSA, and the local RAG corpus.

**Fix recipe (5 min when needed):**
1. In `hackerone_client.py`, decode the token: `token = base64.b64decode(HACKERONE_TOKEN).decode('utf-8')`
2. Register a personal HackerOne profile on the account
3. Re-test: `python test_api_keys.py`

---

## Related Skills

- `@rag-query-interface` — technical ChromaDB setup and query syntax details
- `@hacktricks-client` — HackTricks API integration (45/50 weekly calls remaining as of 2026-10-30)
- `@github-tool-discovery` — GitHub API tool/advisory search

---

## References

- ChromaDB docs: https://docs.trychroma.com/
- all-MiniLM-L6-v2: https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2
- PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings
- EvilWAF: https://github.com/nicowillis/EvilWAF
- WAFNinja: https://github.com/nicowillis/WAFNinja (no license — use at own risk)
- MASTER-REPORT.md: `03-KNOWLEDGE-BASE\04-MASTER-REPORT\MASTER-REPORT.md` (1,189 lines, tools 5–10)

*Authored: 2026-10-31 | Seat-40 knowledge-integrator | Next index refresh: after Phase B (tools 1–4) completion*
