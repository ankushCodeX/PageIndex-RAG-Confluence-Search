# SKILL: PageIndex Engine (Vectorless Search)
## Copilot Skill — src/engines/pageindex.js

### PURPOSE
Implement the PageIndex algorithm — a vectorless, graph-based relevance search using:
1. Confluence REST API full-text search (CQL)
2. BM25-style keyword scoring
3. Page link graph traversal (pages that link to/from highly relevant pages get boosted)
4. LLM-based final reranking (using your small internal model)

This is the "new" algorithm to benchmark against RAG.

### WHY PageIndex?
- No vector DB needed — works with raw Confluence API
- Leverages Confluence's internal link structure as a relevance signal
- Like PageRank but for your knowledge base
- Small LLM does final reranking (cheap, fast)

### ALGORITHM FLOW
```
Query Input
    ↓
[Step 1] CQL Search → Get top 20 candidate pages from Confluence
    ↓
[Step 2] BM25 Scoring → Score each page title+excerpt against query
    ↓
[Step 3] Link Graph Boost → Pages linked-to by top results get +bonus
    ↓
[Step 4] LLM Rerank → Small model picks top N from scored candidates
    ↓
Results Output (same interface as RAG engine)
```

---

## PROMPT FOR COPILOT

```
Create src/engines/pageindex.js — a vectorless Confluence search engine.

### STEP 1: CQL Search via Confluence REST API

async function confluenceSearch(query, spaceKey, maxCandidates = 20) {
  // Build CQL query string
  // CQL: text ~ "query terms" AND space = "SPACEKEY" ORDER BY lastModified DESC
  // API: GET /wiki/rest/api/content/search?cql=...&limit=20&expand=body.view,metadata.labels,children.page,ancestors
  // Auth: Basic auth with base64(user:token) or Bearer token
  // Return array of raw pages with: id, title, url, body excerpt, lastModified, ancestors, children count
  
  Use axios. Build the URL carefully:
  const cql = `text ~ "${query}" ${spaceKey ? `AND space = "${spaceKey}"` : ''} ORDER BY lastModified DESC`;
  const url = `${config.CONFLUENCE_BASE_URL}/rest/api/content/search`;
  const params = { cql, limit: maxCandidates, expand: 'body.view,version,ancestors' };
  const auth = Buffer.from(`${config.CONFLUENCE_USER}:${config.CONFLUENCE_TOKEN}`).toString('base64');
  
  Extract from response: results[].id, title, _links.webui (page URL), body.view.value (HTML content → strip tags for text)
}

### STEP 2: BM25 Scoring

function bm25Score(query, document, options = { k1: 1.5, b: 0.75 }) {
  // Simplified BM25 for title + content
  // 1. Tokenize query and document (split on spaces, lowercase, remove stopwords)
  // 2. For each query term: calculate term frequency in doc, document frequency approximation
  // 3. BM25 formula: sum of IDF * (tf * (k1+1)) / (tf + k1 * (1 - b + b * (docLen/avgDocLen)))
  // 4. Title match bonus: if query terms appear in title, multiply score by 1.5
  // 5. Return normalized score 0.0 to 1.0
  
  Stopwords list: ['the','a','an','is','it','in','on','at','to','for','of','and','or','but','how','what','why','when']
}

### STEP 3: Link Graph Traversal

async function getLinkBoost(topPageIds, allPages) {
  // For the top 5 pages by BM25, fetch their child pages and parent pages
  // Any page in allPages that is a child/parent of a top-5 page gets +0.1 boost
  // This simulates PageRank: well-connected neighbors of relevant pages are likely relevant
  
  // API: GET /wiki/rest/api/content/{id}/child/page
  // Only do this for top 5 pages to keep it fast (parallel with Promise.all)
  // Return a Map<pageId, boostScore>
  
  Keep timeout at 3 seconds max — if Confluence is slow, skip boost and log warning
}

### STEP 4: LLM Reranking

async function llmRerank(query, candidates, maxResults = 5) {
  // Send top 10 scored candidates to small LLM for final reranking
  // Prompt: "Given this search query: '{query}', rank these Confluence pages by relevance.
  //          Return ONLY a JSON array of page IDs in order of relevance, most relevant first.
  //          Pages: {JSON of id, title, excerpt for each candidate}"
  // Parse JSON response, reorder candidates array, return top maxResults
  
  // If LLM fails or times out: fall back to BM25 ordering (graceful degradation)
  // LLM call must complete in < 5 seconds, otherwise skip reranking
  
  Use config.LLM_API_URL and config.LLM_API_TOKEN
  Model: config.LLM_MODEL
  max_tokens: 200 (we only need a short JSON array back)
}

### MAIN SEARCH FUNCTION

async function search(query, options = {}) {
  const { maxResults = 5, spaceKey = null, timeout = 15000 } = options;
  const startTime = Date.now();
  
  try {
    // Step 1: Get candidates from Confluence CQL
    const candidates = await confluenceSearch(query, spaceKey, 20);
    if (candidates.length === 0) return emptyResult(startTime);
    
    // Step 2: BM25 score each candidate
    const scored = candidates.map(page => ({
      ...page,
      bm25Score: bm25Score(query, page.title + ' ' + page.textContent)
    })).sort((a, b) => b.bm25Score - a.bm25Score);
    
    // Step 3: Link graph boost (parallel, non-blocking)
    const topIds = scored.slice(0, 5).map(p => p.id);
    const boostMap = await getLinkBoost(topIds, scored).catch(() => new Map());
    const boosted = scored.map(p => ({
      ...p,
      finalScore: Math.min(1.0, p.bm25Score + (boostMap.get(p.id) || 0))
    })).sort((a, b) => b.finalScore - a.finalScore);
    
    // Step 4: LLM rerank top 10
    const reranked = await llmRerank(query, boosted.slice(0, 10), maxResults);
    
    // Step 5: Normalize to standard interface
    return {
      engine: 'pageindex',
      results: reranked.slice(0, maxResults).map(normalizeResult),
      queryTime: Date.now() - startTime,
      error: null,
      metadata: { candidatesFound: candidates.length, linkBoostApplied: boostMap.size > 0 }
    };
  } catch (err) {
    return { engine: 'pageindex', results: [], queryTime: Date.now() - startTime, error: err.message };
  }
}

### HELPER: normalizeResult
Convert internal page object to standard interface:
{
  pageId: page.id,
  title: page.title,
  url: config.CONFLUENCE_BASE_URL + page._links.webui,
  excerpt: page.textContent.slice(0, 300).trim() + '...',
  score: page.finalScore,
  scoreLabel: scoreToLabel(page.finalScore),
  metadata: { lastModified: page.version.when, linkBoosted: boostMap.has(page.id) }
}

module.exports = { search, testConnection };
```

---

## TEST FILE PROMPT

```
Create tests/test-pageindex.js:

const { search } = require('../src/engines/pageindex');

async function main() {
  const query = process.argv[2] || 'API documentation';
  const space = process.argv[3] || null;
  
  console.log(`🔗 Testing PageIndex Engine`);
  console.log(`Query: "${query}"${space ? ` in space: ${space}` : ''}\n`);
  
  const result = await search(query, { maxResults: 5, spaceKey: space });
  
  if (result.error) {
    console.error('❌ Error:', result.error);
    return;
  }
  
  console.log(`✅ Found ${result.results.length} results in ${result.queryTime}ms`);
  console.log(`   Candidates searched: ${result.metadata?.candidatesFound}`);
  console.log(`   Link boost applied: ${result.metadata?.linkBoostApplied}\n`);
  
  result.results.forEach((r, i) => {
    console.log(`[${i+1}] ${r.title}`);
    console.log(`    Score: ${r.score.toFixed(3)} (${r.scoreLabel})`);
    console.log(`    ${r.url}`);
    console.log(`    ${r.excerpt.slice(0, 120)}...\n`);
  });
}

main().catch(console.error);
```

Run with: `node tests/test-pageindex.js "deployment guide" ENG`

---

## VALIDATION CHECKLIST
- [ ] CQL search returns pages from Confluence
- [ ] BM25 scoring produces different scores per page (not all the same)
- [ ] Link boost runs without crashing (even if no links found)
- [ ] LLM reranking works OR gracefully falls back to BM25 order
- [ ] `node tests/test-pageindex.js "onboarding"` returns results
- [ ] Error case: wrong Confluence token → returns error object, doesn't crash
