# 🚀 Confluence AI Search — Hackathon Master Plan
## Dual Engine: RAG (existing) + Real PageIndex (local, open-source)

---

## 🎯 What You're Building

A **single-page application** where a user types a question → both engines race simultaneously → a side-by-side comparison shows which algorithm found better Confluence pages and why.

**The WOW moment at demo**: Two different AI philosophies, live head-to-head.

---

## 🧠 How Real PageIndex Works (Know This Before You Code)

PageIndex is a **Python library** from `github.com/VectifyAI/PageIndex`. It does two things:

### Phase 1: INDEXING (done ONCE, offline, before your app runs)
```
Your 50-60 Confluence pages
    ↓  (export to Markdown — one .md file per page)
run_pageindex.py --md_path page.md
    ↓  (calls your LLM API to generate summaries)
results/page_structure.json   ← hierarchical tree index
```
The tree JSON looks like:
```json
{
  "title": "API Integration Guide",
  "node_id": "0001",
  "start_index": 0,
  "end_index": 5,
  "summary": "This section covers how to...",
  "nodes": [
    { "title": "Authentication", "node_id": "0002", "summary": "...", "nodes": [] },
    { "title": "Endpoints", "node_id": "0003", "summary": "...", "nodes": [...] }
  ]
}
```

### Phase 2: RETRIEVAL (happens on every user query, at runtime)
```
User query: "how do we handle auth tokens?"
    ↓
LLM reasons over the tree JSON:
  "Root node summary mentions auth → go deeper → Authentication node"
    ↓
LLM returns relevant node_ids
    ↓
Your code maps node_id → confluence page URL + content excerpt
```

**Key insight**: No vector math. The LLM reads the tree, reasons like a human reading a table of contents, and drills down to the right section.

---

## 🗂️ Project Structure

```
confluence-ai-search/
├── pageindex/                     ← Clone of VectifyAI/PageIndex repo (downloaded)
│   ├── page_index.py
│   ├── page_index_md.py
│   └── config.yaml
├── run_pageindex.py               ← PageIndex CLI (from repo)
├── requirements.txt               ← PageIndex's requirements
│
├── scripts/
│   ├── 1_export_confluence.py     ← Step 1: Pull pages from Confluence → .md files
│   ├── 2_index_all_pages.py       ← Step 2: Run PageIndex on all .md files
│   └── 3_build_search_index.py    ← Step 3: Merge all tree JSONs → master_index.json
│
├── data/
│   ├── confluence_pages/          ← Raw markdown exports (one .md per Confluence page)
│   │   ├── page_12345.md
│   │   └── ...
│   └── index/
│       ├── page_12345_structure.json   ← Per-page tree from PageIndex
│       └── master_index.json           ← All pages merged + metadata
│
├── src/
│   ├── engines/
│   │   ├── rag_engine.py          ← Calls your existing RAG API endpoint
│   │   └── pageindex_engine.py    ← Runs tree search using your LLM API
│   ├── orchestrator.py            ← Runs both in parallel
│   ├── judge.py                   ← Picks winner
│   └── server.py                  ← Flask/FastAPI server
│
├── public/
│   └── index.html                 ← SPA frontend
│
├── .env.example
└── README.md
```

---

## ⏱️ 10-Hour Sprint

### PRE-WORK (before clock starts, ~1hr):
- Clone PageIndex repo locally
- Verify pip install works for requirements
- Test Confluence API token with one page fetch
- Test your LLM API endpoint with a simple call

### Day 1 (5 hrs): Data Pipeline + Engines

| Hour | Task | Skill File |
|------|------|------------|
| 0–1 | Export Confluence pages → Markdown files | SKILL_1_CONFLUENCE_EXPORT.md |
| 1–2.5 | Run PageIndex indexing on all pages → JSON trees | SKILL_2_PAGEINDEX_INDEXING.md |
| 2.5–3 | Build master_index.json from all trees | SKILL_2_PAGEINDEX_INDEXING.md |
| 3–4 | PageIndex retrieval engine (tree search via LLM) | SKILL_3_PAGEINDEX_ENGINE.md |
| 4–5 | RAG adapter + Orchestrator + Judge | SKILL_4_ORCHESTRATOR.md |

### Day 2 (5 hrs): Server + UI

| Hour | Task | Skill File |
|------|------|------------|
| 5–6 | Flask server + REST API | SKILL_4_ORCHESTRATOR.md |
| 6–8 | Frontend SPA | SKILL_5_FRONTEND.md |
| 8–9 | End-to-end test + polish | SKILL_6_MAS_PLUGIN.md |
| 9–10 | Demo prep + README | — |

---

## 📦 Dependencies (Enterprise / Nexus Check)

### PageIndex's requirements.txt (from repo):
```
openai          ← Used as HTTP client; you'll redirect to your internal LLM
pymupdf         ← PDF parsing (NOT needed if you use --md_path only)
PyPDF2          ← PDF parsing (NOT needed if you use --md_path only)
tiktoken        ← Token counting
python-dotenv   ← .env file loading
requests        ← HTTP calls
```

### Your additions:
```
flask           ← Lightweight server (or fastapi + uvicorn)
```

### IMPORTANT — Enterprise Strategy:
- Use `--md_path` (Markdown mode) ONLY → skip `pymupdf` and `PyPDF2` entirely
- PageIndex uses `openai` Python SDK but you'll override `base_url` to point to your internal LLM
- `tiktoken` may need Nexus — verify ahead of time. If unavailable, use `len(text.split()) * 1.3` as token estimate
- `flask` is universally available on Nexus

---

## 🔌 LLM API Override Strategy

PageIndex uses the `openai` Python SDK internally. Since you have an internal LLM with OpenAI-compatible API:

```python
# In your .env:
CHATGPT_API_KEY=your-internal-token
OPENAI_API_BASE=http://your-internal-llm/v1  # override base URL

# In your code, before calling PageIndex:
import openai
openai.api_key = os.getenv("CHATGPT_API_KEY")
openai.base_url = os.getenv("OPENAI_API_BASE")  # redirect to your LLM
```

PageIndex reads `CHATGPT_API_KEY` from `.env` — matching the env var name is all you need.

---

## 🔄 MAS Plugin Interface (Output Contract)

```python
# Both engines return this standard shape:
{
    "engine": "pageindex",          # or "rag"
    "results": [
        {
            "page_id": "12345",
            "title": "API Integration Guide",
            "url": "https://confluence.org.com/pages/12345",
            "excerpt": "This section covers authentication...",
            "score": 0.87,          # 0.0-1.0
            "score_label": "HIGH",  # HIGH / MEDIUM / LOW
            "node_path": ["Root", "Auth", "Tokens"],  # PageIndex tree path
            "metadata": {}
        }
    ],
    "query_time_ms": 1240,
    "error": None
}
```
