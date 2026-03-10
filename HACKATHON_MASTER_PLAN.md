# 🚀 Confluence AI Search — Vibe Coding Hackathon
## Master Plan: RAG + PageIndex Dual-Engine Search

---

## 🎯 Project Vision

A **single-page application** that lets users type a natural language query, searches Confluence using **two AI algorithms simultaneously** (RAG Vector Search + PageIndex Link-based search), and presents results side-by-side — letting users see which approach wins for their query.

**Wow Factor**: Live head-to-head battle between search strategies with confidence scores, result explanations, and a winner declaration.

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│              Confluence AI Search SPA                │
│  ┌─────────────┐         ┌─────────────────────────┐│
│  │  Query Input │────────▶│    Search Orchestrator  ││
│  └─────────────┘         └──────────┬──────────────┘│
│                               ┌─────┴─────┐          │
│                         ┌─────▼─┐     ┌───▼──────┐  │
│                         │  RAG  │     │PageIndex │  │
│                         │Engine │     │  Engine  │  │
│                         └─────┬─┘     └───┬──────┘  │
│                               └─────┬─────┘          │
│                          ┌──────────▼──────────┐     │
│                          │  Results Renderer    │     │
│                          │  + Winner Judge      │     │
│                          └─────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

**Backend**: Node.js/Express (enterprise-safe, available on Nexus)  
**Frontend**: Vanilla HTML/CSS/JS (zero framework risk, zero Nexus issues)  
**LLM**: Your org API endpoint (small model for reranking/explanation)  
**Vector DB**: Your existing RAG endpoint  
**PageIndex**: Confluence REST API + BM25 + LLM relevance scoring  

---

## ⏱️ 10-Hour Sprint Plan

### Day 1 (5 hrs): Core Engines

| Hour | Task | Deliverable |
|------|------|-------------|
| 0–1 | Project scaffold + config | `server.js`, `.env.example`, `package.json` |
| 1–2 | RAG adapter (connect to your existing endpoint) | `src/engines/rag.js` |
| 2–3 | PageIndex engine (Confluence REST + BM25) | `src/engines/pageindex.js` |
| 3–4 | Search orchestrator (parallel execution) | `src/orchestrator.js` |
| 4–5 | Basic API test + Postman collection | All engines testable independently |

### Day 2 (5 hrs): UI + Polish

| Hour | Task | Deliverable |
|------|------|-------------|
| 5–6 | Frontend SPA shell + search bar | `public/index.html` |
| 6–7 | Dual results panel with animations | Side-by-side result cards |
| 7–8 | Winner judge + score visualization | `src/judge.js` + score bars |
| 8–9 | Integration test + demo data | End-to-end working |
| 9–10 | Demo polish + README | Hackathon-ready |

---

## 📦 Project Structure

```
confluence-ai-search/
├── src/
│   ├── engines/
│   │   ├── rag.js              ← Your existing RAG endpoint adapter
│   │   └── pageindex.js        ← NEW: PageIndex engine
│   ├── orchestrator.js         ← Runs both engines in parallel
│   ├── judge.js                ← Scores and picks winner
│   └── config.js               ← All env vars in one place
├── public/
│   ├── index.html              ← Full SPA (single file)
│   └── favicon.ico
├── skills/                     ← Copilot skill files (this package)
│   ├── SKILL_RAG_ADAPTER.md
│   ├── SKILL_PAGEINDEX.md
│   ├── SKILL_ORCHESTRATOR.md
│   ├── SKILL_JUDGE.md
│   └── SKILL_FRONTEND.md
├── tests/
│   ├── test-rag.js
│   ├── test-pageindex.js
│   └── test-orchestrator.js
├── .env.example
├── package.json
├── server.js
└── README.md
```

---

## 🔌 MAS Plugin Interface

This project exposes a **standard MCP-compatible interface** so it plugs into your Multi-Agent System:

```javascript
// Input contract (MAS-compatible)
{
  "query": "string",           // User's natural language query
  "engines": ["rag", "pageindex"], // Which engines to use
  "maxResults": 5,             // Per engine
  "spaceKey": "MYSPACE",       // Optional Confluence space filter
  "returnFormat": "mcp"        // "mcp" | "json" | "html"
}

// Output contract
{
  "results": {
    "rag": [...],
    "pageindex": [...],
    "winner": "rag|pageindex|tie",
    "confidence": 0.87
  },
  "metadata": { "queryTime": 1200, "ragScore": 0.91, "pageIndexScore": 0.84 }
}
```

---

## 🧪 Independent Testing

Each engine is independently testable:

```bash
# Test RAG engine alone
node tests/test-rag.js --query "deployment process" --space ENG

# Test PageIndex engine alone  
node tests/test-pageindex.js --query "deployment process" --space ENG

# Test orchestrator (both engines)
node tests/test-orchestrator.js --query "deployment process"

# Full server
npm start
curl -X POST http://localhost:3000/api/search \
  -H "Content-Type: application/json" \
  -d '{"query": "deployment process", "engines": ["rag","pageindex"]}'
```
