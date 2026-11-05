---
name: rag-query-interface
description: General RAG interface documentation for the ChromaDB vector database backing the pentest knowledge base. Covers collection setup, embedding model details, query_texts syntax, distance score interpretation, incremental ingestion, and index management. Use when setting up a new RAG index, debugging poor retrieval quality, adding new sources, or migrating the index to a new host.
triggers:
  - RAG query
  - semantic search knowledge base
  - vector search
  - ChromaDB query
  - chromadb setup
  - RAG index
  - embedding query
  - vector database pentest
  - chroma collection
  - ingest knowledge base
  - add documents to RAG
  - rebuild RAG index
---

# RAG QUERY INTERFACE — ChromaDB Technical Skill

## Overview

This skill documents the technical setup, query interface, collection structure, and maintenance procedures for the ChromaDB vector database that powers the pentest knowledge base. It is the infrastructure layer under `@knowledge-agent`.

**Current state (verified 2026-10-30/31):**
- ChromaDB version: **1.5.9** (installed and verified on PC and both VPSes)
- Embedding model: **all-MiniLM-L6-v2** (ONNX, 79.3 MB, cached locally)
- Collection name: `pentest-kb-2026`
- Index location: `03-KNOWLEDGE-BASE/02-AUTOMATED-SOURCES/rag-index/chroma.sqlite3`
- Index state: **Initialized but empty** — PayloadsAllTheThings/SecLists not yet ingested
- VPS index location: `/srv/dr/projects/pentest-kb/rag-index/`

---

## Architecture

```
Source documents (.md files, CVE tables, technique guides)
    │
    ▼ langchain-text-splitters
Chunked documents (512 token chunks, 64 token overlap)
    │
    ▼ sentence-transformers (all-MiniLM-L6-v2, ONNX)
Embedding vectors (384-dimensional float32)
    │
    ▼
ChromaDB 1.5.9 (local SQLite backend)
    Collection: pentest-kb-2026
    File: rag-index/chroma.sqlite3
    │
    ▼ query_texts=[user_query]
Top-K ranked results with distance scores
    │
    ▼
rag_client.py response (JSON: score, source, text)
```

---

## ChromaDB Setup

### Installation

```bash
# All dependencies
pip install chromadb>=0.5.0 sentence-transformers>=3.0.0 langchain-text-splitters>=0.2.0

# Verify installation
python3 -c "import chromadb; print(chromadb.__version__)"
# Expected: 1.5.9 (or higher)

python3 -c "from sentence_transformers import SentenceTransformer; print('OK')"
```

### Collection Initialization

```python
import chromadb
from chromadb.utils import embedding_functions

# Initialize persistent client
client = chromadb.PersistentClient(
    path="/srv/dr/projects/pentest-kb/rag-index"
    # Windows: path="C:\\Users\\rabim\\OneDrive\\Desktop\\NEW RESULTS\\03-KNOWLEDGE-BASE\\02-AUTOMATED-SOURCES\\rag-index"
)

# Embedding function (all-MiniLM-L6-v2)
ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)

# Create or get existing collection
collection = client.get_or_create_collection(
    name="pentest-kb-2026",
    embedding_function=ef,
    metadata={"hnsw:space": "cosine"}  # cosine distance for semantic similarity
)

print(f"Collection: {collection.name} | Documents: {collection.count()}")
```

### Distance Metric

The collection uses **cosine distance** (`hnsw:space: cosine`).

ChromaDB returns `distance` scores where:
- 0.0 = perfect match (identical vectors)
- 1.0 = completely orthogonal (no similarity)
- Values above 1.0 = anti-correlated (rare with text)

The `rag_client.py` converts distances to similarity scores: `score = 1 - distance`

**Score interpretation:**
```
score ≥ 0.90  → near-exact match — high confidence, use directly
score 0.75–0.89 → strong match — good; verify against source  
score 0.60–0.74 → moderate match — useful hint; cross-check
score 0.50–0.59 → weak match — low signal; may be noise
score < 0.50   → no useful match — rephrase or rebuild index
```

---

## Query Interface

### Direct ChromaDB Query

```python
import chromadb
from chromadb.utils import embedding_functions

client = chromadb.PersistentClient(path="/srv/dr/projects/pentest-kb/rag-index")
ef = embedding_functions.SentenceTransformerEmbeddingFunction(model_name="all-MiniLM-L6-v2")
collection = client.get_collection(name="pentest-kb-2026", embedding_function=ef)

# Basic query
results = collection.query(
    query_texts=["how to bypass Cloudflare WAF with sqlmap"],
    n_results=5
)

# Parse results
for i, (doc, meta, dist) in enumerate(zip(
    results['documents'][0],
    results['metadatas'][0],
    results['distances'][0]
)):
    score = 1 - dist
    source = meta.get('source', 'unknown')
    print(f"[{score:.3f}] {source}")
    print(f"  {doc[:300]}")
    print()
```

### Query with Metadata Filtering

```python
# Filter by source file
results = collection.query(
    query_texts=["second order SQL injection"],
    n_results=5,
    where={"source": {"$contains": "MASTER-REPORT"}}
)

# Filter by CVE-related content
results = collection.query(
    query_texts=["CVE-2026-19949"],
    n_results=10,
    where={"type": {"$eq": "cve_entry"}}
)

# Exclude low-priority sources
results = collection.query(
    query_texts=["Metasploit meterpreter payload"],
    n_results=5,
    where={"priority": {"$gte": 2}}  # priority: 1=high, 3=low
)
```

### Batch Query (multiple questions at once)

```python
# Batch query — more efficient than multiple single queries
results = collection.query(
    query_texts=[
        "sqlmap tamper scripts cloudflare",
        "ffuf vhost discovery",
        "metasploit meterpreter persistence"
    ],
    n_results=3
)

# results['documents'] is a list of lists — one list per query
for query_idx, (docs, dists) in enumerate(zip(results['documents'], results['distances'])):
    print(f"\n=== Query {query_idx + 1} ===")
    for doc, dist in zip(docs, dists):
        print(f"  [{1-dist:.3f}] {doc[:200]}")
```

---

## Document Ingestion

### Chunking Configuration

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

# Primary: Markdown-aware splitter (preserves section structure)
md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"), ("##", "h2"), ("###", "h3")
    ]
)

# Secondary: Character-level splitter for size control
char_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,    # tokens (approximate — actual tokens depend on tokenizer)
    chunk_overlap=64,  # 12.5% overlap preserves context across chunk boundaries
    separators=["\n\n", "\n", " ", ""]
)

def chunk_markdown(text: str, source_path: str) -> list[dict]:
    """Returns list of {text, metadata} dicts for ingestion."""
    # First split by headers
    header_splits = md_splitter.split_text(text)
    chunks = []
    for split in header_splits:
        # Then split large sections by character count
        sub_chunks = char_splitter.split_text(split.page_content)
        for i, chunk in enumerate(sub_chunks):
            chunks.append({
                "text": chunk,
                "metadata": {
                    "source": source_path,
                    "section": split.metadata.get("h2", split.metadata.get("h1", "unknown")),
                    "chunk_index": i,
                    "total_chunks": len(sub_chunks)
                }
            })
    return chunks
```

### Ingestion Pipeline

```python
import chromadb
from chromadb.utils import embedding_functions
from pathlib import Path
import hashlib

def ingest_file(collection, filepath: str, priority: int = 2):
    """Ingest a single markdown file into the collection."""
    path = Path(filepath)
    text = path.read_text(encoding='utf-8')
    
    chunks = chunk_markdown(text, source_path=str(path))
    
    # Generate stable IDs (hash of source + chunk index)
    ids = [
        hashlib.sha256(f"{filepath}::{i}".encode()).hexdigest()[:16]
        for i in range(len(chunks))
    ]
    
    # Add priority to metadata
    for chunk in chunks:
        chunk['metadata']['priority'] = priority
    
    # Upsert (add or update)
    collection.upsert(
        documents=[c['text'] for c in chunks],
        metadatas=[c['metadata'] for c in chunks],
        ids=ids
    )
    
    print(f"[+] Ingested {len(chunks)} chunks from {path.name}")

# Ingest priority sources
ingest_file(collection, "03-KNOWLEDGE-BASE/04-MASTER-REPORT/MASTER-REPORT.md", priority=1)
ingest_file(collection, "03-KNOWLEDGE-BASE/02-AUTOMATED-SOURCES/TOOL-05-SQLMAP.md", priority=1)
ingest_file(collection, "03-KNOWLEDGE-BASE/02-AUTOMATED-SOURCES/TOOL-06-WPSCAN.md", priority=1)

# Ingest PayloadsAllTheThings (large — use walk)
for md_file in Path("03-KNOWLEDGE-BASE/02-AUTOMATED-SOURCES/PayloadsAllTheThings").rglob("*.md"):
    ingest_file(collection, str(md_file), priority=3)
```

---

## Recommended Ingestion Order

Ingest in this order for best relevance weighting:

| # | Source | Priority | Expected Chunks |
|---|--------|----------|-----------------|
| 1 | `MASTER-REPORT.md` | 1 (highest) | ~400 |
| 2 | `TOOL-05-SQLMAP.md` through `TOOL-10-METASPLOIT.md` | 1 | ~600 total |
| 3 | CVE tables (extracted from API reports) | 2 | ~200 |
| 4 | `PayloadsAllTheThings/` (rglob *.md) | 3 | ~8,000 |
| 5 | `EvilWAF/` | 3 | ~500 |
| 6 | `sqlmap-tamper-collection/` | 2 | ~300 |
| 7 | `Bug-Bounty-Playbook/` | 3 | ~1,000 |
| 8 | `BBLabs-Academy/` | 3 | ~800 |
| 9 | `Intigriti-Blog/` | 3 | ~400 |

**Total estimated chunks after full ingestion:** ~12,000–15,000 documents  
**Expected rebuild time (VPS with cached model):** 3–8 minutes  
**First rebuild (no model cache):** 8–15 minutes (downloads 79.3 MB model)

---

## Index Management

### Status Check

```python
import chromadb
client = chromadb.PersistentClient(path="/srv/dr/projects/pentest-kb/rag-index")
for coll in client.list_collections():
    col = client.get_collection(coll.name)
    print(f"Collection: {coll.name} | Documents: {col.count()}")
```

```bash
# Via rag_client.py
python3 /srv/dr/projects/pentest-kb/rag_client.py --status
# Expected output:
# Collection: pentest-kb-2026 | Documents: 12847
# Last rebuild: 2026-10-31 14:23:00
# Index size: 42.1 MB
```

### Rebuild from Scratch

```bash
# Full rebuild (clears and re-ingests all sources)
python3 /srv/dr/projects/pentest-kb/rag_client.py --rebuild

# Add single file without rebuild
python3 /srv/dr/projects/pentest-kb/rag_client.py --add-file /path/to/new-technique.md
```

### Delete and Recreate Collection

```python
client = chromadb.PersistentClient(path="/srv/dr/projects/pentest-kb/rag-index")
# Nuclear option — only when index is corrupt
client.delete_collection("pentest-kb-2026")
# Then re-initialize and re-ingest
```

---

## VPS Deployment

### Web-dev VPS (179.61.138.8)

```bash
# Deploy rag_client.py and source files
ssh rabi@179.61.138.8

# Install dependencies
pip3 install chromadb>=0.5.0 sentence-transformers>=3.0.0 langchain-text-splitters>=0.2.0

# Create project directory
mkdir -p /srv/dr/projects/pentest-kb/rag-index

# On PC: SCP the files
scp "C:\Users\rabim\OneDrive\Desktop\NEW RESULTS\03-KNOWLEDGE-BASE\03-KNOWLEDGE-AGENT\rag_client.py" \
    rabi@179.61.138.8:/srv/dr/projects/pentest-kb/

# Run rebuild (first time: downloads model ~79 MB, then builds index)
python3 /srv/dr/projects/pentest-kb/rag_client.py --rebuild
```

### Knowledge VPS (179.61.251.98)

```bash
# Same procedure on second VPS
ssh rabi@179.61.251.98
mkdir -p /srv/dr/projects/pentest-kb/rag-index
# ... same install and deploy steps
python3 /srv/dr/projects/pentest-kb/rag_client.py --rebuild
```

---

## Troubleshooting

### Poor retrieval quality (scores < 0.5)

1. Check document count: `python3 rag_client.py --status` — if 0, index is empty → ingest first
2. Rephrase query: use technique keywords from the source documents
3. Rebuild index: source documents may have changed → `python3 rag_client.py --rebuild`
4. Check chunk size: if chunks are too large (>1024 tokens), embeddings average-out meaning

### "Collection not found" error

```python
# Always use get_or_create, never get
collection = client.get_or_create_collection(name="pentest-kb-2026", embedding_function=ef)
```

### Slow first query (30–60 seconds)

Model loading from disk. Subsequent queries use the cached model: 532–727 ms typical.

### Index too large / disk space

```bash
# Check index size
du -sh /srv/dr/projects/pentest-kb/rag-index/

# ChromaDB vacuum (removes deleted documents)
python3 -c "
import chromadb
client = chromadb.PersistentClient('/srv/dr/projects/pentest-kb/rag-index')
# Vacuum is automatic in 1.x; no manual call needed
print('OK')
"
```

### ONNX model download fails (no internet on VPS)

```bash
# Pre-download on PC, then scp to VPS
# Model cache location (Linux): ~/.cache/torch/sentence_transformers/
# Model cache location (Windows): C:\Users\rabim\.cache\torch\sentence_transformers\

# SCP the cached model
scp -r ~/.cache/torch/sentence_transformers/sentence-transformers_all-MiniLM-L6-v2 \
    rabi@179.61.138.8:~/.cache/torch/sentence_transformers/
```

---

## Quality Gate

Before claiming the index is production-ready:

```bash
# Test 1: CVE entry retrieval
python3 rag_client.py --query "CVE-2026-19949 second order SQL injection WordPress migration"
# Must return MASTER-REPORT.md result with score ≥ 0.70

# Test 2: Tool technique retrieval
python3 rag_client.py --query "ffuf vhost discovery Host header fuzzing subdomain"
# Must return TOOL-07-FFUF.md result with score ≥ 0.70

# Test 3: Payload retrieval
python3 rag_client.py --query "Metasploit meterpreter reverse TCP staged payload windows"
# Must return TOOL-10-METASPLOIT.md result with score ≥ 0.70
```

All three must score ≥ 0.70 for the index to be considered production-quality. Scores of 0.50–0.69 indicate the index needs more content ingested.

---

## Related Skills

- `@knowledge-agent` — higher-level query interface (use this for day-to-day queries)
- `@hacktricks-client` — HackTricks API (for technique lookup beyond local corpus)
- `@github-tool-discovery` — GitHub tool and advisory search

---

## References

- ChromaDB documentation: https://docs.trychroma.com/
- ChromaDB GitHub: https://github.com/chroma-core/chroma
- sentence-transformers: https://www.sbert.net/
- all-MiniLM-L6-v2 model card: https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2
- langchain-text-splitters: https://python.langchain.com/docs/how_to/split_by_markdown_header/
- rag_client.py: `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/rag_client.py`
- Index path: `03-KNOWLEDGE-BASE/02-AUTOMATED-SOURCES/rag-index/chroma.sqlite3`
- KNOWLEDGE-AGENT-SPEC.md: `03-KNOWLEDGE-BASE/03-KNOWLEDGE-AGENT/KNOWLEDGE-AGENT-SPEC.md`

*Authored: 2026-10-31 | Seat-40 knowledge-integrator | ChromaDB 1.5.9 verified live*
