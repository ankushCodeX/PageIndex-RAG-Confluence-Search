# SKILL: Frontend SPA — Dual Engine Search UI
## Copilot Skill — public/index.html

### PURPOSE
Create a single-file, zero-dependency SPA that presents the dual-engine Confluence search
as a dramatic head-to-head battle. Visually impressive for hackathon demo.

### DESIGN DIRECTION
- **Theme**: Command center / mission control aesthetic
- **Color palette**: Dark background (#0d1117), RAG in electric blue (#00d4ff), PageIndex in emerald (#00ff88), Winner gold (#ffd700)
- **Typography**: Monospace for scores/data, clean sans-serif for content
- **Key WOW moment**: Both engines animate in simultaneously, then the winner is revealed with a crown animation
- **Layout**: Split-panel with center "VS" divider, verdict banner at top

---

## PROMPT FOR COPILOT

```
Create public/index.html — a complete single-file SPA (HTML + CSS + JS, no external dependencies except Google Fonts).

### LAYOUT STRUCTURE:
┌──────────────────────────────────────────────────────┐
│  🔍 CONFLUENCE AI SEARCH  [Space: ENG ▼]            │
│  ┌────────────────────────────────────────────────┐  │
│  │  Type your question or topic...    [SEARCH]    │  │
│  └────────────────────────────────────────────────┘  │
├──────────────────┬───┬──────────────────────────────┤
│   ⚡ RAG         │VS │      🔗 PageIndex             │
│   Vector Search  │   │      Link Graph Search        │
│                  │   │                               │
│  [Result cards]  │   │  [Result cards]               │
│                  │   │                               │
│  ████ 87/100     │   │  ████ 72/100                  │
├──────────────────┴───┴──────────────────────────────┤
│  🏆 RAG WINS — Semantic search excelled here...      │
└──────────────────────────────────────────────────────┘

### REQUIRED FEATURES:

1. SEARCH BAR
   - Large, prominent text input with placeholder "What would you like to find in Confluence?"
   - Space selector dropdown (populated via GET /api/spaces)
   - SEARCH button + Enter key support
   - Character count / query length indicator
   - Loading spinner with "Dispatching both engines..." text during search

2. RESULTS PANELS (side-by-side, equal width)
   - RAG panel: electric blue (#00d4ff) accent
   - PageIndex panel: emerald green (#00ff88) accent  
   - Each panel shows: engine name, query time badge, result cards
   - Result cards: title (clickable link), score bar (colored), excerpt text, OPEN button
   - Score bar animates filling up from 0 to actual score on reveal
   - If engine errored: show red error state with message

3. SCORE VISUALIZATION
   - Below each panel: horizontal score bar showing judge score (/100)
   - Breakdown bars for: Result Count, Avg Relevance, Top Relevance, Speed
   - Animate these bars filling up after results appear

4. WINNER VERDICT BANNER
   - Appears at top after results load
   - Background gradient matching winner's color
   - Shows: crown emoji, "RAG WINS" / "PAGEINDEX WINS" / "TIE"
   - Confidence indicator: "Decisive Win" / "Close Match" / "Draw"
   - One-sentence explanation from judge

5. QUERY HISTORY
   - Last 5 searches shown below search bar
   - Click to re-run
   - Store in localStorage

6. COPY/EXPORT BUTTON
   - Top right: "Export Results" button
   - Downloads JSON of the full API response
   - Useful for MAS integration testing

### JAVASCRIPT BEHAVIOR:

async function performSearch(query) {
  // Show loading state on both panels simultaneously
  // POST to /api/search
  // Parse response
  // Animate results in (stagger 100ms per card)
  // Animate score bars
  // Reveal winner banner with CSS animation
  // Add to query history
}

function renderResultCard(result, engineColor) {
  // Returns HTML string for a result card
  // Title: clickable link to Confluence page
  // Score bar: colored fill based on engine, width = score * 100%
  // Score label badge: HIGH/MEDIUM/LOW
  // Excerpt: first 200 chars with "..." 
}

function renderScoreBreakdown(breakdown, engineId) {
  // Mini bar chart for each judging criterion
}

function revealWinner(verdict) {
  // Animate winner banner sliding in from top
  // Add 'winner' CSS class to winning panel (glow effect)
  // If tie: both panels get gold glow
}

### CSS REQUIREMENTS:
- CSS custom properties for theming
- Dark background: #0d1117
- Card backgrounds: #161b22 with subtle border
- RAG blue: #00d4ff
- PageIndex green: #00ff88  
- Winner gold: #ffd700
- Smooth transitions on all score bars (transition: width 0.8s ease-out)
- Winner panel gets box-shadow glow in its color
- Mobile responsive: stack panels vertically on < 768px width
- Monospace font for scores, 'Segoe UI' for content (no Google Fonts — enterprise safe)

### ERROR STATES:
- Network error: "Could not reach the search server" with retry button
- Engine error: Show error badge on that engine's panel, still show other engine's results
- No results: "No pages found. Try broader terms." with search tips

### DEMO MODE:
- Add ?demo=true URL parameter support
- In demo mode, use hardcoded mock responses instead of real API calls
- This allows UI/UX testing without backend
- Mock data should look realistic (real Confluence-style titles and excerpts)

Generate the complete index.html — all HTML, CSS, and JavaScript in one file, no external dependencies except system fonts.
```

---

## VALIDATION CHECKLIST
- [ ] Page loads without JS errors in browser console
- [ ] Search returns and renders dual results
- [ ] Score bars animate correctly
- [ ] Winner banner appears with right color/text
- [ ] Clicking a result title opens Confluence page in new tab
- [ ] Works in demo mode (`?demo=true`) without backend
- [ ] Responsive on mobile viewport
- [ ] Export JSON button downloads valid JSON file
