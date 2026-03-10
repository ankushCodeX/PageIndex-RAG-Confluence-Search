# SKILL: Project Scaffold & Configuration
## Copilot Skill — confluence-ai-search

### PURPOSE
Generate the complete project scaffold for the Confluence AI Search dual-engine application.
This is the FIRST skill to run. It creates package.json, server.js, config.js, and .env.example.

### ENTERPRISE CONSTRAINTS
- All npm packages must be available in Nexus (standard packages only — express, axios, cors, dotenv, node-fetch)
- No exotic dependencies
- Node.js 16+ compatible
- No TypeScript (keep it simple for hackathon)

---

## PROMPT FOR COPILOT

```
Generate a Node.js/Express project scaffold for "confluence-ai-search" with these files:

### 1. package.json
{
  "name": "confluence-ai-search",
  "version": "1.0.0",
  "description": "Dual-engine Confluence search: RAG + PageIndex",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js",
    "test:rag": "node tests/test-rag.js",
    "test:pageindex": "node tests/test-pageindex.js",
    "test:all": "node tests/test-orchestrator.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "axios": "^1.6.0",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1"
  }
}

### 2. .env.example (copy to .env and fill in values)
# Confluence Config
CONFLUENCE_BASE_URL=https://your-org.atlassian.net/wiki
CONFLUENCE_TOKEN=your-confluence-api-token
CONFLUENCE_USER=your-email@org.com
CONFLUENCE_DEFAULT_SPACE=ENG

# RAG Engine (your existing endpoint)
RAG_API_URL=http://your-rag-service/api/search
RAG_API_TOKEN=your-rag-token

# LLM Config (small model for reranking)
LLM_API_URL=http://your-internal-llm/v1/chat/completions
LLM_API_TOKEN=your-llm-token
LLM_MODEL=your-small-model-name

# Server
PORT=3000
NODE_ENV=development

### 3. src/config.js
Load all env vars with validation. Export a config object.
Throw clear error messages if required vars are missing.
Required: CONFLUENCE_BASE_URL, CONFLUENCE_TOKEN, RAG_API_URL
Optional with defaults: PORT=3000, LLM_MODEL="default"

### 4. server.js
Express server with:
- CORS enabled
- JSON body parser
- Static file serving from /public folder
- Route: POST /api/search  → orchestrator
- Route: GET /api/health   → returns {status:"ok", engines:["rag","pageindex"]}
- Route: GET /api/spaces   → lists available Confluence spaces
- Error handler middleware
- Console log on startup showing URL and available engines

### 5. Create folder structure:
mkdir -p src/engines src tests public
```

---

## VALIDATION CHECKLIST
After running this skill, verify:
- [ ] `npm install` succeeds (all packages on Nexus)
- [ ] `node server.js` starts without errors
- [ ] `curl http://localhost:3000/api/health` returns `{"status":"ok"}`
- [ ] `.env` created from `.env.example` with real values filled in
