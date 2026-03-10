# ⚡ Hackathon Execution Guide
## Step-by-Step: From Zero to Demo in 10 Hours

---

## PRE-HACKATHON CHECKLIST (Do this BEFORE the clock starts)

```bash
# Verify these work BEFORE hackathon day:
□ node --version                    # Need 16+
□ npm install express               # Confirm Nexus access
□ curl YOUR_RAG_ENDPOINT/health     # Your existing RAG API is up
□ curl YOUR_CONFLUENCE_URL/wiki/rest/api/space  # Confluence API accessible
□ Postman or curl available for testing
□ VS Code + Copilot extension active
□ All skill files loaded into Copilot context (or accessible)
```

---

## DAY 1 — HOUR BY HOUR

### ⏰ HOUR 0 (0:00–1:00): Project Scaffold

**Copilot Prompt:**
```
Load SKILL_1_SCAFFOLD.md and generate the complete project scaffold for confluence-ai-search.
Create all files listed. Use npm packages: express, axios, cors, dotenv only.
```

**Manual steps:**
```bash
mkdir confluence-ai-search && cd confluence-ai-search
# After Copilot generates files:
cp .env.example .env
# Fill in .env with real values
npm install
node server.js
curl http://localhost:3000/api/health
# Should return: {"status":"ok","engines":["rag","pageindex"]}
```

**✅ Done when:** Health endpoint returns 200

---

### ⏰ HOUR 1–2 (1:00–2:00): RAG Adapter

**Copilot Prompt:**
```
Load SKILL_2_RAG_ADAPTER.md. Create src/engines/rag.js adapting my existing RAG endpoint.
My endpoint shape is: [PASTE YOUR ACTUAL RESPONSE SHAPE HERE]
```

**Test:**
```bash
node tests/test-rag.js "how to deploy our application"
```

**✅ Done when:** Test returns 3+ results with title, url, score fields

---

### ⏰ HOUR 2–3 (2:00–3:00): PageIndex Engine

**Copilot Prompt:**
```
Load SKILL_3_PAGEINDEX.md. Create src/engines/pageindex.js.
My Confluence base URL is [URL], auth is token-based.
Skip LLM reranking step if LLM is unavailable — fallback to BM25 order.
```

**Test:**
```bash
node tests/test-pageindex.js "onboarding guide"
```

**If Confluence API is slow:** Reduce `maxCandidates` to 10 and skip link graph step (comment it out)

**✅ Done when:** Test returns results (even if different from RAG results)

---

### ⏰ HOUR 3–4 (3:00–4:00): Orchestrator + Judge

**Copilot Prompt:**
```
Load SKILL_4_ORCHESTRATOR.md. Create src/orchestrator.js and src/judge.js.
Wire into server.js POST /api/search route.
```

**Test:**
```bash
node tests/test-orchestrator.js "API integration guide"
# Should show both engines, scores, and winner
```

**✅ Done when:** Both engines run in parallel and verdict shows winner

---

### ⏰ HOUR 4–5 (4:00–5:00): End-to-End API Test

```bash
# Full API test:
curl -X POST http://localhost:3000/api/search \
  -H "Content-Type: application/json" \
  -d '{"query": "how to set up LDAP authentication", "maxResults": 3}'

# Expected: JSON with results.rag, results.pageindex, verdict
```

**Fix any issues, then STOP and get a good night's sleep. Day 1 done! 🎉**

---

## DAY 2 — HOUR BY HOUR

### ⏰ HOUR 5–6 (5:00–6:00): Frontend Shell

**Copilot Prompt:**
```
Load SKILL_5_FRONTEND.md. Create public/index.html — complete single-file SPA.
Dark theme, command-center aesthetic. Make the dual panel layout first,
search functionality second, animations third.
```

**Open browser:** http://localhost:3000
**Test demo mode:** http://localhost:3000?demo=true (no API needed)

**✅ Done when:** Page loads and demo mode shows mock results in both panels

---

### ⏰ HOUR 6–7 (6:00–7:00): Wire Frontend to Backend

**Add real search:**
```javascript
// In index.html JS:
async function performSearch(query) {
  const response = await fetch('/api/search', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({ query, maxResults: 5 })
  });
  const data = await response.json();
  renderResults(data);
}
```

**Test end-to-end:** Type a real query, see real results from both engines

**✅ Done when:** Real Confluence results appear in UI

---

### ⏰ HOUR 7–8 (7:00–8:00): Score Animations + Winner Banner

**Copilot Prompt:**
```
Add CSS animations to the results: score bars fill up on load, 
winner panel gets a glowing border, verdict banner slides in from top.
Keep it impressive but not overdone.
```

**✅ Done when:** The demo looks impressive enough to present

---

### ⏰ HOUR 8–9 (8:00–9:00): MAS Plugin + End-to-End

**Copilot Prompt:**
```
Load SKILL_6_MAS_PLUGIN.md. Create src/mas-plugin.js and add MCP routes to server.js.
Also create src/cli.js for standalone testing.
```

**Test MAS plugin:**
```bash
node src/cli.js --query "CI/CD pipeline" --format json
curl -X POST localhost:3000/mcp/tools/confluence_search \
  -d '{"query":"test"}' -H "Content-Type:application/json"
```

**✅ Done when:** CLI works and MCP endpoint returns valid schema

---

### ⏰ HOUR 9–10 (9:00–10:00): Polish + Demo Prep

**Final checklist:**
```bash
□ README.md written (5 min, ask Copilot to generate from skills)
□ .env.example has all variables documented
□ Demo script prepared (3 test queries that make both engines look different)
□ http://localhost:3000?demo=true works without backend (for offline demo)
□ node src/cli.js --query "test" works (MAS integration demo)
□ One slide/screenshot showing the architecture
```

**Best demo queries to prepare:**
1. Conceptual query: `"explain our microservices architecture"` → RAG likely wins
2. Keyword query: `"JIRA ticket creation API endpoint"` → PageIndex likely wins  
3. Ambiguous query: `"best practices"` → usually a tie, shows balance

---

## 🆘 EMERGENCY FALLBACKS

| Problem | Fallback |
|---------|----------|
| RAG endpoint is down | Switch to PageIndex-only mode in demo |
| Confluence API rate limited | Use demo mode with cached responses |
| LLM unavailable | Skip LLM reranking step in PageIndex |
| Both engines slow | Add `?timeout=30000` and show "processing..." animation |
| Nexus package missing | Use CDN in dev, document for production |

---

## 🎤 DEMO SCRIPT (3 minutes)

> "We built a Confluence search that uses TWO AI algorithms simultaneously and lets you see which one is smarter for your query.
>
> On the left: RAG — our existing vector embeddings approach. Semantic meaning. Understands intent.
> On the right: PageIndex — our new algorithm. No vectors needed. Uses keyword matching + page link graph traversal.
>
> [Type query] Watch them race... [pause for results]
>
> RAG wins here — makes sense, this is a conceptual query. Now let me try a specific API question...
> [Type technical query] Now PageIndex wins — exact keyword matching is better here.
>
> The beauty? It's pluggable. Any agent in our MAS system can call this as a single tool via MCP and get the best result. Enterprise-safe, runs on Nexus dependencies only, and the entire thing is independently testable."

---

## 📊 Success Metrics for Judges

- ✅ Both engines return results (not just one)
- ✅ Live comparison visible in UI  
- ✅ Winner verdict with explanation (not just scores)
- ✅ Different queries produce different winners (shows real difference)
- ✅ MAS integration demo works via CLI
- ✅ All dependencies from Nexus (enterprise-ready)
