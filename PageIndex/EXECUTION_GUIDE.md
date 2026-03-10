# ⚡ Hackathon Execution Guide
## Step-by-Step: Confluence AI Search in 10 Hours

---

## 🔴 PRE-HACKATHON PREP (Do TONIGHT — 1 hour)

This will save you 2 hours on the day. Do not skip.

```bash
# 1. Clone PageIndex repo into your project
git clone https://github.com/VectifyAI/PageIndex.git confluence-ai-search
cd confluence-ai-search

# 2. Install requirements (verify Nexus access!)
pip install -r requirements.txt
# If tiktoken fails → note it, you'll patch it
# If pymupdf fails → it's OK, you'll use --md_path only (no PDF)

# 3. Verify your LLM API works
curl -X POST http://your-internal-llm/v1/chat/completions \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"your-model","messages":[{"role":"user","content":"Say OK"}],"max_tokens":5}'
# Must return {"choices":[{"message":{"content":"OK"}}]}

# 4. Verify Confluence API works
curl -u "your.email:YOUR_TOKEN" \
  "https://yourorg.atlassian.net/wiki/rest/api/content?spaceKey=ENG&limit=1"
# Must return JSON with results array

# 5. Create .env from template
cp .env.example .env
# Fill in ALL values — this is the most important pre-work

# 6. Create project folders
mkdir -p src/engines data/confluence_pages data/index/trees public tests scripts

# 7. Install Flask
pip install flask flask-cors
```

**✅ Pre-work done when**: LLM API returns, Confluence API returns, .env filled

---

## DAY 1 — HOURS 0-5

### ⏰ HOUR 0-1: Export Confluence Pages

**Copilot prompt:**
```
Load SKILL_1_CONFLUENCE_EXPORT.md
Generate scripts/1_export_confluence.py
My Confluence uses token-based auth (Bearer token, not Basic auth)
```

**Run it:**
```bash
python scripts/1_export_confluence.py
ls data/confluence_pages/   # should show .md files
cat data/confluence_pages/page_XXXXX.md   # verify it's readable markdown
```

**Acceptance**: 50+ `.md` files, each starting with `# Title`, readable content

---

### ⏰ HOUR 1-2.5: Index All Pages with PageIndex

**Copilot prompt:**
```
Load SKILL_2_PAGEINDEX_INDEXING.md
Generate:
  - pageindex_llm_patch.py
  - scripts/2_index_all_pages.py
  - scripts/3_build_search_index.py
My LLM API base URL is: [paste your URL]
My model name is: [paste your model name]
```

**Run indexing** (this takes 10-20 minutes for 50-60 pages):
```bash
python scripts/2_index_all_pages.py
# Watch it: should print per-page progress
# LLM calls happening: ~2-3 calls per page
```

**While indexing runs** (background): Write the RAG adapter (Hour 3).

```bash
# After indexing completes:
python scripts/3_build_search_index.py
# Should print: "Pages indexed: 52, Total nodes: 187"

# Quick verify:
python -c "
import json
d = json.load(open('data/index/master_index.json'))
print('Pages:', d['total_pages'])
print('Nodes:', d['total_nodes'])
print('Sample page:', d['pages'][0]['title'])
print('Sample node:', d['all_nodes'][5]['node_title'])
"
```

**If tiktoken fails:**
```python
# Add this to pageindex_llm_patch.py after apply_llm_patch():
import sys
# Mock tiktoken if not available
try:
    import tiktoken
except ImportError:
    class FakeTiktoken:
        @staticmethod
        def encoding_for_model(m): return FakeTiktoken()
        def encode(self, text): return list(range(int(len(text.split()) * 1.3)))
    sys.modules['tiktoken'] = FakeTiktoken()
```

---

### ⏰ HOUR 2.5-3: PageIndex Engine + RAG Adapter

**Copilot prompt:**
```
Load SKILL_3_PAGEINDEX_ENGINE.md
Generate src/engines/pageindex_engine.py
```

**Copilot prompt:**
```
Load SKILL_4_ORCHESTRATOR.md (RAG Adapter section only)
Generate src/engines/rag_engine.py
My RAG API response shape is: [paste one actual response from your RAG API]
```

**Test individually:**
```bash
python tests/test_pageindex_engine.py "how to authenticate"
python -c "from src.engines.rag_engine import search; print(search('authentication'))"
```

---

### ⏰ HOUR 3-4: Orchestrator + Judge

**Copilot prompt:**
```
Load SKILL_4_ORCHESTRATOR.md (Orchestrator + Judge + Flask Server sections)
Generate: src/orchestrator.py, src/judge.py, src/server.py
```

**Test:**
```bash
python src/server.py &
curl http://localhost:5000/api/health
curl -X POST localhost:5000/api/search \
  -H "Content-Type:application/json" \
  -d '{"query":"how to deploy our service"}'
# Should return both engine results + verdict
```

**✅ DAY 1 DONE when**: The API returns real results from both engines. REST now.

---

## DAY 2 — HOURS 5-10

### ⏰ HOUR 5-7: Frontend

**Copilot prompt:**
```
Load SKILL_5_FRONTEND.md
Generate public/index.html
Make sure PageIndex results show the tree path (🌲 Root → Section → Subsection)
and LLM reasoning text. This is the key visual differentiator from RAG.
```

**Test demo mode first (no backend needed):**
Open `public/index.html` directly in browser with `?demo=true`
→ Should show mock results with tree paths

**Wire to backend:**
```bash
python src/server.py
```
Open `http://localhost:5000` → search something real

---

### ⏰ HOUR 7-8: Polish + Integration Testing

**Prepare 3 demo queries** (test these, see which engine wins each):

1. **Conceptual query**: `"explain our microservices architecture"` → RAG likely wins
2. **Specific/structural query**: `"step by step guide to onboard new engineers"` → PageIndex likely wins (it navigates document structure)
3. **Factual lookup**: `"what port does our API gateway run on"` → often a tie

**Test each query, screenshot winner + results for demo script**

**MAS plugin test:**
```bash
# Standalone CLI test
python -c "
import sys; sys.path.insert(0, '.')
from src.orchestrator import search
import json
result = search('authentication guide', {'return_format': 'mcp'})
print(json.dumps(result, indent=2))
"

# HTTP MCP test
curl localhost:5000/mcp/tools
curl -X POST localhost:5000/mcp/tools/confluence_search \
  -H "Content-Type:application/json" \
  -d '{"query":"deployment process","maxResults":3}'
```

---

### ⏰ HOUR 8-9: Demo Polish

**Verify demo mode works offline:**
```
http://localhost:5000?demo=true
```
This is your backup if the live API is slow during demo.

**Write README** (ask Copilot):
```
Generate README.md for confluence-ai-search project.
Include: setup steps, how to run indexing, how to start server,
API endpoints, how to plug into MAS via MCP endpoint.
Keep it concise — engineering audience.
```

---

### ⏰ HOUR 9-10: Final Checks + Demo Script

**Final checklist:**
```
□ python scripts/1_export_confluence.py        → works
□ python scripts/2_index_all_pages.py          → works (or already done)
□ python scripts/3_build_search_index.py       → works
□ python tests/test_pageindex_engine.py "test" → returns results
□ python src/server.py                         → starts clean
□ http://localhost:5000                        → loads SPA
□ http://localhost:5000?demo=true              → works offline
□ curl /api/health                             → {status: ok, index_loaded: true}
□ curl /api/search with real query             → both engines return results
□ curl /mcp/tools/confluence_search            → MCP format response
□ Different queries → different winners        → proves real comparison
```

---

## 🎤 3-Minute Demo Script

> "We built a Confluence search that runs TWO AI algorithms simultaneously and lets you see which one is smarter for your specific query.
>
> On the left: **RAG** — our existing vector embedding search. It works by converting text to numbers and finding what's *numerically similar*.
>
> On the right: **PageIndex** — an open-source algorithm from VectifyAI. No vectors, no embeddings. Instead, it builds a **tree of contents** from each Confluence page using an LLM, and then *reasons* through that tree like a human expert reading a document.
>
> [Type query 1: conceptual]
> Watch them race... RAG wins here. Conceptual queries suit semantic similarity.
>
> [Point to PageIndex result]
> But notice what PageIndex shows that RAG can't: the **reasoning path**.
> *Guide → Setup → Authentication → Token Management* — this is the LLM literally navigating the document like a human would. RAG gives you a number. PageIndex gives you *why*.
>
> [Type query 2: structural]
> Now a structural question... PageIndex wins this one. Step-by-step guides have clear document structure, and tree traversal finds that structure precisely.
>
> The entire thing is enterprise-safe: only PyPI packages, redirected to our internal LLM, all data stays on-prem. And it's MCP-compatible — [open CLI] — any agent in our MAS system can call this as a tool and get the winning result automatically."

---

## 🆘 Emergency Fallbacks

| Problem | Fix |
|---------|-----|
| PageIndex indexing hangs | Kill it, run on 10 pages only: `ls data/confluence_pages/ | head -10 | xargs ...` |
| tiktoken not on Nexus | Add mock in `pageindex_llm_patch.py` (see Hour 1-2.5 section) |
| LLM API too slow for tree search | Reduce `top_pages_to_search` to 2 |
| RAG endpoint is down | Demo PageIndex-only mode: `{"engines":["pageindex"]}` |
| Confluence API rate limited | Use already-exported .md files, re-run indexing only |
| Both engines slow at demo | Pre-cache 3 demo query results in a JSON file, serve from cache |
