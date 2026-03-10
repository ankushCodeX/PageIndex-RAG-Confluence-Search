# SKILL 5: Frontend SPA — Dual Engine Search UI
## public/index.html (single file, no external dependencies)

### PURPOSE
A dramatic, visually impressive single-page app showing the two AI engines racing.
The PageIndex "tree path" visualization is the unique differentiator — show the LLM's
reasoning trail through the document tree, which RAG can't show.

### DESIGN DIRECTION
- **Aesthetic**: Dark terminal/command-center. Like a war room where two AI agents compete.
- **Colors**: #0a0e1a background, RAG in cyan #00d4ff, PageIndex in lime #7fff00, gold #ffd700 for winner
- **Typography**: `Courier New` for scores and paths (monospace), clean system sans-serif for content
- **WOW moments**:
  1. Both panels animate in simultaneously when search fires
  2. PageIndex results show the TREE PATH: `Guide → Setup → Authentication → Tokens` — this is unique, RAG can't do this
  3. Winner banner slides down with crown + color explosion
  4. Score bars fill up with a race animation

---

## PROMPT FOR COPILOT

```
Create public/index.html — complete single-file SPA (HTML + CSS + JS, no external dependencies).

### PAGE STRUCTURE

<header>
  <div class="logo">⚡ Confluence AI Search</div>
  <div class="engine-badges">
    <span class="badge rag">RAG Vector</span>
    <span class="vs">VS</span>
    <span class="badge pageindex">PageIndex Tree</span>
  </div>
</header>

<main>
  <section class="search-section">
    <input id="query-input" type="text" placeholder="Ask anything about your Confluence knowledge base..." />
    <button id="search-btn">SEARCH</button>
    <div id="query-history"><!-- last 5 queries as chips --></div>
  </section>

  <section id="verdict-banner" class="hidden">
    <!-- Winner announcement bar: slides down after results load -->
    <span class="crown">👑</span>
    <span id="winner-text">RAG WINS</span>
    <span id="winner-score-diff"></span>
    <span id="winner-explanation"></span>
  </section>

  <section class="results-area">
    <div class="engine-panel" id="rag-panel">
      <div class="panel-header">
        <h2>⚡ RAG</h2>
        <span class="subtitle">Vector Similarity Search</span>
        <span id="rag-time" class="time-badge">--</span>
      </div>
      <div id="rag-results" class="results-list">
        <div class="waiting">Waiting for query...</div>
      </div>
      <div class="score-bar-container">
        <label>Judge Score</label>
        <div class="score-track"><div id="rag-score-fill" class="score-fill rag-color" style="width:0%"></div></div>
        <span id="rag-score-num">--/100</span>
      </div>
      <div id="rag-breakdown" class="breakdown hidden"></div>
    </div>

    <div class="divider">VS</div>

    <div class="engine-panel" id="pageindex-panel">
      <div class="panel-header">
        <h2>🌲 PageIndex</h2>
        <span class="subtitle">Reasoning-Based Tree Search</span>
        <span id="pi-time" class="time-badge">--</span>
      </div>
      <div id="pageindex-results" class="results-list">
        <div class="waiting">Waiting for query...</div>
      </div>
      <div class="score-bar-container">
        <label>Judge Score</label>
        <div class="score-track"><div id="pi-score-fill" class="score-fill pi-color" style="width:0%"></div></div>
        <span id="pi-score-num">--/100</span>
      </div>
      <div id="pi-breakdown" class="breakdown hidden"></div>
    </div>
  </section>
</main>

### CSS REQUIREMENTS

:root {
  --bg: #0a0e1a;
  --card-bg: #111827;
  --border: #1f2937;
  --rag-color: #00d4ff;
  --pi-color: #7fff00;       /* lime green — distinct from RAG */
  --winner-gold: #ffd700;
  --text: #e5e7eb;
  --muted: #6b7280;
  --font-mono: 'Courier New', monospace;
  --font-ui: -apple-system, 'Segoe UI', sans-serif;
}

body: background var(--bg), color var(--text), font-family var(--font-ui)

.engine-panel: 
  - background var(--card-bg)
  - border 1px solid var(--border)
  - border-radius 12px
  - flex: 1
  - transition: box-shadow 0.4s ease

.engine-panel.winner:
  - box-shadow: 0 0 30px rgba(winner-color, 0.4)
  - border-color: winner-color

Results area: display flex, gap 20px (stack vertically on < 768px)

.result-card:
  - padding 12px 16px
  - border-left 3px solid engine-color
  - background rgba(255,255,255,0.03)
  - margin-bottom 8px
  - opacity 0 initially → fade in with animation-delay staggered 100ms per card

.tree-path: (UNIQUE to PageIndex results)
  - font-family var(--font-mono)
  - font-size 0.75rem
  - color var(--pi-color)
  - opacity 0.8
  - format: "Guide → Setup → Auth → Tokens"
  - This is the key differentiator — show LLM reasoning visually!

.score-fill:
  - height 8px
  - border-radius 4px
  - transition: width 0.9s cubic-bezier(0.4, 0, 0.2, 1)
  - starts at width 0%, animated to actual score%

Verdict banner:
  - Slides down from top with translateY(-100%) → 0 animation
  - Background gradient in winner's color (semi-transparent)
  - Border-bottom 2px solid winner-color

Loading state:
  - Show animated dots: "Searching. · Searching.. · Searching..." cycling

### JAVASCRIPT

const API_BASE = "";  // same origin

let searchHistory = JSON.parse(localStorage.getItem("searchHistory") || "[]");

async function performSearch(query) {
  if (!query || query.trim().length < 2) return;
  
  // 1. Clear previous results, show loading state in both panels
  showLoading("rag-results");
  showLoading("pageindex-results");
  document.getElementById("verdict-banner").classList.add("hidden");
  document.getElementById("rag-panel").classList.remove("winner");
  document.getElementById("pageindex-panel").classList.remove("winner");
  
  // 2. Call API
  const response = await fetch(`${API_BASE}/api/search`, {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({ query: query.trim(), max_results: 5 })
  });
  
  if (!response.ok) {
    showError("rag-results", "Search failed");
    showError("pageindex-results", "Search failed");
    return;
  }
  
  const data = await response.json();
  
  // 3. Render results (staggered animation)
  renderEngineResults("rag", data.results.rag, data.verdict);
  renderEngineResults("pageindex", data.results.pageindex, data.verdict);
  
  // 4. Animate score bars
  setTimeout(() => {
    animateScoreBar("rag", data.verdict.scores?.rag || 0);
    animateScoreBar("pi", data.verdict.scores?.pageindex || 0);
  }, 300);
  
  // 5. Show verdict banner (after short delay for drama)
  setTimeout(() => showVerdict(data.verdict), 800);
  
  // 6. Highlight winning panel
  if (data.verdict.winner !== "tie") {
    setTimeout(() => {
      document.getElementById(`${data.verdict.winner}-panel`).classList.add("winner");
    }, 1000);
  }
  
  // 7. Save to history
  addToHistory(query);
}

function renderEngineResults(engineKey, engineData, verdict) {
  const container = document.getElementById(`${engineKey === "pageindex" ? "pageindex" : engineKey}-results`);
  
  if (engineData?.error) {
    container.innerHTML = `<div class="error-state">❌ ${engineData.error}</div>`;
    return;
  }
  
  const results = engineData?.results || [];
  if (results.length === 0) {
    container.innerHTML = `<div class="empty-state">No results found</div>`;
    return;
  }
  
  container.innerHTML = results.map((r, i) => `
    <div class="result-card" style="animation-delay: ${i * 100}ms">
      <div class="result-title">
        <a href="${r.url}" target="_blank">${r.title}</a>
        <span class="score-badge ${r.score_label.toLowerCase()}">${r.score_label}</span>
      </div>
      ${engineKey === "pageindex" && r.node_path?.length > 1 ? `
        <div class="tree-path">🌲 ${r.node_path.join(" → ")}</div>
        ${r.reasoning ? `<div class="reasoning">💭 ${r.reasoning}</div>` : ""}
      ` : ""}
      <div class="excerpt">${r.excerpt?.slice(0, 200)}${r.excerpt?.length > 200 ? "..." : ""}</div>
      <div class="result-footer">
        <span class="score-num">${(r.score * 100).toFixed(0)}% relevant</span>
        <a class="open-link" href="${r.url}" target="_blank">Open ↗</a>
      </div>
    </div>
  `).join("");
  
  document.getElementById(`${engineKey === "rag" ? "rag" : "pi"}-time`).textContent = 
    `${engineData.query_time_ms}ms`;
}

function showVerdict(verdict) {
  const banner = document.getElementById("verdict-banner");
  const winnerText = document.getElementById("winner-text");
  const explanation = document.getElementById("winner-explanation");
  
  const labels = { rag: "⚡ RAG WINS", pageindex: "🌲 PAGEINDEX WINS", tie: "🤝 TIE MATCH" };
  winnerText.textContent = labels[verdict.winner] || "NO RESULT";
  explanation.textContent = verdict.explanation || "";
  
  // Set banner color based on winner
  banner.className = `verdict-banner ${verdict.winner}`;
  banner.classList.remove("hidden");
}

function animateScoreBar(prefix, score) {
  document.getElementById(`${prefix}-score-fill`).style.width = `${score}%`;
  document.getElementById(`${prefix}-score-num`).textContent = `${score}/100`;
}

// Enter key support
document.getElementById("query-input").addEventListener("keypress", e => {
  if (e.key === "Enter") performSearch(e.target.value);
});
document.getElementById("search-btn").addEventListener("click", () => {
  performSearch(document.getElementById("query-input").value);
});

// Load history chips on page load
renderHistory();

### DEMO MODE (?demo=true in URL)
If URLSearchParams has demo=true, use mock data instead of API:
- Mock: 3 rag results and 3 pageindex results with realistic Confluence titles
- PageIndex mock results MUST include node_path and reasoning fields
- Verdict: pageindex wins (score 78 vs 62) — shows the tree path advantage
- This allows UI testing without backend

### ERROR HANDLING
- fetch fails: show red "Could not reach server" state in both panels, with Retry button
- Only one engine fails: show that engine's error state, other engine shows normally
- Query too short: shake the input box (CSS animation), don't call API

Generate the complete index.html — all HTML, CSS, and JS in a single file.
No external scripts, no Google Fonts (use system fonts only for enterprise compliance).
```

---

## VALIDATION CHECKLIST
- [ ] `http://localhost:5000?demo=true` works without backend — shows mock results
- [ ] PageIndex results show `🌲 Path → To → Node` (this is the key visual differentiator)
- [ ] PageIndex results show `💭 reasoning text` from LLM
- [ ] Score bars animate smoothly after results load
- [ ] Winner banner slides in with correct color
- [ ] Winning panel gets glow border
- [ ] Mobile: panels stack vertically on narrow viewport
- [ ] Enter key in input triggers search
- [ ] Error state: kill the server and search → shows error UI, doesn't crash JS
