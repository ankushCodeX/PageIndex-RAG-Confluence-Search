# SKILL 4: RAG Adapter + Orchestrator + Judge + Flask Server
## src/engines/rag_engine.py + src/orchestrator.py + src/judge.py + src/server.py

### PURPOSE
Wire your existing RAG endpoint into the same interface as PageIndex, run both in parallel,
and expose everything as a REST API.

---

## PROMPT FOR COPILOT — RAG Adapter

```
Create src/engines/rag_engine.py

Wraps the existing RAG Vector DB API endpoint.
Uses only: requests, os, time, json (standard + requests)

import requests, os, time, json

RAG_API_URL = os.getenv("RAG_API_URL")          # Your existing endpoint
RAG_API_TOKEN = os.getenv("RAG_API_TOKEN")       # Bearer token
RAG_DEFAULT_SPACE = os.getenv("CONFLUENCE_SPACE_KEY", "")

### STANDARD RESULT SHAPE (same as PageIndex engine)
{
    "page_id": str,
    "title": str,
    "url": str,
    "excerpt": str,
    "score": float,          # 0.0-1.0
    "score_label": str,      # HIGH / MEDIUM / LOW
    "node_path": [],         # Empty list for RAG (no tree path)
    "metadata": {}
}

def search(query, options=None):
    """
    Call existing RAG endpoint.
    Normalize response to standard shape.
    Handle both common response shapes:
      Shape A: {"results": [{"id", "title", "url", "content", "score"}]}
      Shape B: {"matches": [{"pageId", "pageTitle", "pageUrl", "text", "similarity"}]}
    
    Score normalization:
    - Cosine similarity (0-1): use as-is
    - Cosine distance (0-1): score = 1 - distance
    - If score > 1: normalize to 0-1 by dividing by max score in result set
    
    score_label: score >= 0.8 → HIGH, >= 0.5 → MEDIUM, else LOW
    
    Return: {engine: "rag", results: [...], query_time_ms: int, error: None or str}
    """

def test_connection():
    """Try a minimal search call, return True if successful."""
```

---

## PROMPT FOR COPILOT — Orchestrator

```
Create src/orchestrator.py

Runs both engines concurrently using threading (not asyncio — keep it simple).
Uses only: threading, time, json, os (standard library only)

import threading, time

def search(query, options=None):
    """
    options: {
        engines: ["rag", "pageindex"],    # which engines to run
        max_results: 5,
        return_format: "json"             # "json" or "mcp"
    }
    
    Run selected engines in PARALLEL using threads:
    1. Start a thread for each engine
    2. Collect results from threads (timeout: 30s per engine)
    3. If a thread fails/times out → return error result for that engine
    4. Pass results to judge.evaluate()
    5. Return combined response
    
    THREAD PATTERN:
    results = {}
    def run_rag():
        results["rag"] = rag_engine.search(query, options)
    def run_pageindex():
        results["pageindex"] = pageindex_engine.search(query, options)
    
    threads = [threading.Thread(target=run_rag), threading.Thread(target=run_pageindex)]
    for t in threads: t.start()
    for t in threads: t.join(timeout=30)
    
    RESPONSE SHAPE:
    {
        "query": query,
        "total_time_ms": int,
        "results": {
            "rag": { engine result dict },
            "pageindex": { engine result dict }
        },
        "verdict": { winner, confidence, scores, breakdown, explanation },
        "metadata": { engines_run, timestamp }
    }
    
    MCP FORMAT (when return_format == "mcp"):
    {
        "type": "tool_result",
        "tool": "confluence_search",
        "content": {
            "query": str,
            "winner": str,
            "confidence": float,
            "top_results": [...],   # top 3 from winning engine
            "all_results": {...},
            "query_time_ms": int
        }
    }
    """
```

---

## PROMPT FOR COPILOT — Judge

```
Create src/judge.py

Evaluates both engine results and declares a winner.
Uses only standard library.

def evaluate(results, query):
    """
    Scoring weights (adjust based on what you want to highlight at demo):
    
    1. Result count (15%): More results = better
    2. Average score (40%): Average relevance score across results (LLM-assigned for PageIndex, vector for RAG)
    3. Top score (30%): Highest single result score  
    4. Speed (15%): Faster engine wins (inverted: lower ms = higher score, benchmark 5000ms)
    
    Per-engine score = weighted sum, scaled to 0-100
    
    Winner rules:
    - Difference < 8 points: "tie"
    - Otherwise: higher scorer wins
    
    Confidence = abs(score_diff) / 40, clamped to 0.0-1.0
    
    Explanation strings:
    - RAG wins: "Vector RAG wins by {diff} pts. Semantic embedding excelled at understanding the conceptual meaning of this query."
    - PageIndex wins: "PageIndex wins by {diff} pts. Reasoning-based tree navigation found more structurally precise content."
    - Tie: "Both engines performed similarly ({rag_score} vs {pageindex_score}). This query had strong signals for both approaches."
    
    Return:
    {
        "winner": "rag" | "pageindex" | "tie" | "none",
        "confidence": 0.0-1.0,
        "scores": {"rag": int, "pageindex": int},
        "breakdown": {
            "rag": {"result_count": int, "avg_relevance": int, "top_relevance": int, "speed": int, "query_time_ms": int},
            "pageindex": {...}
        },
        "explanation": str
    }
    """
```

---

## PROMPT FOR COPILOT — Flask Server

```
Create src/server.py

Flask REST API server.
Dependencies: flask, os, json (flask + standard library only)

from flask import Flask, request, jsonify
from flask_cors import CORS  # if available, otherwise add CORS headers manually
import os, json

app = Flask(__name__, static_folder="../public", static_url_path="")

# Serve SPA at root
@app.route("/")
def index():
    return app.send_static_file("index.html")

# Health check
@app.route("/api/health")
def health():
    return jsonify({
        "status": "ok",
        "engines": ["rag", "pageindex"],
        "index_loaded": _check_index_ready()
    })

# Main search endpoint
@app.route("/api/search", methods=["POST"])
def search():
    body = request.get_json()
    if not body or not body.get("query"):
        return jsonify({"error": "query is required"}), 400
    
    query = body["query"].strip()
    if len(query) < 2:
        return jsonify({"error": "query too short"}), 400
    
    from src.orchestrator import search as orchestrate
    result = orchestrate(query, {
        "engines": body.get("engines", ["rag", "pageindex"]),
        "max_results": min(body.get("max_results", 5), 10),
        "return_format": body.get("return_format", "json")
    })
    return jsonify(result)

# List indexed pages (useful for demo)
@app.route("/api/pages")
def list_pages():
    from pathlib import Path
    import json as j
    meta_file = Path("data/index/pages_metadata.json")
    if not meta_file.exists():
        return jsonify({"error": "Not indexed yet"}), 404
    pages = j.loads(meta_file.read_text())
    return jsonify({"total": len(pages), "pages": pages[:20]})  # first 20 only

# MCP tool endpoint
@app.route("/mcp/tools/confluence_search", methods=["POST"])
def mcp_search():
    body = request.get_json()
    from src.orchestrator import search as orchestrate
    result = orchestrate(body.get("query", ""), {
        "engines": body.get("engines", ["rag", "pageindex"]),
        "max_results": body.get("maxResults", 5),
        "return_format": "mcp"
    })
    return jsonify(result)

# MCP tool schema
@app.route("/mcp/tools")
def mcp_tools():
    return jsonify([{
        "name": "confluence_search",
        "description": "Search Confluence using RAG + PageIndex dual-engine AI",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Natural language search query"},
                "engines": {"type": "array", "items": {"type": "string"}, "default": ["rag","pageindex"]},
                "maxResults": {"type": "integer", "default": 5, "minimum": 1, "maximum": 10}
            },
            "required": ["query"]
        }
    }])

def _check_index_ready():
    from pathlib import Path
    return Path("data/index/master_index.json").exists()

if __name__ == "__main__":
    port = int(os.getenv("PORT", 5000))
    print(f"🚀 Confluence AI Search server running at http://localhost:{port}")
    print(f"   Index ready: {_check_index_ready()}")
    app.run(host="0.0.0.0", port=port, debug=os.getenv("NODE_ENV") == "development")
```

---

## .ENV TEMPLATE PROMPT

```
Create .env.example with all required variables:

# ── Confluence ──────────────────────────────────────────────
CONFLUENCE_BASE_URL=https://yourorg.atlassian.net/wiki
CONFLUENCE_USER=your.email@org.com
CONFLUENCE_TOKEN=your_confluence_api_token
CONFLUENCE_SPACE_KEY=ENG

# ── Your Internal LLM API (OpenAI-compatible) ───────────────
LLM_API_BASE_URL=http://your-internal-llm/v1
LLM_API_TOKEN=your_internal_llm_token
LLM_MODEL_NAME=your-model-name

# ── PageIndex (reads CHATGPT_API_KEY — set to your LLM token)
CHATGPT_API_KEY=your_internal_llm_token   # same as LLM_API_TOKEN

# ── Your Existing RAG Endpoint ───────────────────────────────
RAG_API_URL=http://your-rag-service/api/search
RAG_API_TOKEN=your_rag_token

# ── Server ────────────────────────────────────────────────────
PORT=5000
```

---

## VALIDATION CHECKLIST
- [ ] `python src/server.py` starts without errors
- [ ] `curl http://localhost:5000/api/health` → `{"status":"ok","engines":["rag","pageindex"],"index_loaded":true}`
- [ ] `curl -X POST localhost:5000/api/search -H "Content-Type:application/json" -d '{"query":"authentication"}'` returns both engine results + verdict
- [ ] MCP endpoint: `curl localhost:5000/mcp/tools` returns tool schema
- [ ] If PageIndex engine fails, RAG still returns results (no 500 error)
