# SKILL: RAG Engine Adapter
## Copilot Skill — src/engines/rag.js

### PURPOSE
Adapt your EXISTING RAG Vector DB API endpoint into a standardized engine interface.
This skill wraps your current endpoint — NO changes to the existing RAG system needed.

### INTERFACE CONTRACT
Every engine in this project must conform to this interface:
```javascript
// Input
{ query: string, maxResults: number, spaceKey?: string }

// Output
{
  engine: "rag",
  results: [
    {
      pageId: string,
      title: string,
      url: string,
      excerpt: string,       // Relevant text snippet
      score: number,         // 0.0 to 1.0 relevance score
      scoreLabel: string,    // "HIGH" | "MEDIUM" | "LOW"
      metadata: {}           // Any extra data
    }
  ],
  queryTime: number,         // milliseconds
  error: null | string
}
```

---

## PROMPT FOR COPILOT

```
Create src/engines/rag.js — an adapter for an existing RAG Vector DB API endpoint.

Requirements:
1. Export an async function: search(query, options = {})
2. options: { maxResults=5, spaceKey=null, timeout=10000 }
3. Call the RAG API using axios with Authorization Bearer token from config
4. Handle these response shapes (your existing API might return either):
   Shape A: { results: [{ id, title, url, content, score }] }
   Shape B: { matches: [{ pageId, pageTitle, pageUrl, text, similarity }] }
   Normalize BOTH shapes into the standard interface above
5. Score normalization: if scores are cosine similarity (0-1), keep as-is.
   If scores are distances, convert: score = 1 - distance
6. Add scoreLabel based on score: >=0.8 HIGH, >=0.5 MEDIUM, <0.5 LOW
7. If spaceKey provided, add it as a query param to the RAG API call
8. Wrap in try/catch — on error return { engine:"rag", results:[], error: message, queryTime }
9. Log query time using Date.now() before and after the API call
10. Add a testConnection() export that hits a health endpoint or makes a minimal query

CODE STRUCTURE:
const axios = require('axios');
const config = require('../config');

async function search(query, options = {}) { ... }
async function testConnection() { ... }
function normalizeResult(raw) { ... }
function scoreToLabel(score) { ... }

module.exports = { search, testConnection };
```

---

## TEST FILE PROMPT

```
Create tests/test-rag.js:

const { search, testConnection } = require('../src/engines/rag');

async function main() {
  console.log('🔍 Testing RAG Engine...\n');
  
  // Test 1: Connection
  const connected = await testConnection();
  console.log(connected ? '✅ RAG API connected' : '❌ RAG API unreachable');
  
  // Test 2: Basic search
  const query = process.argv[2] || 'deployment process';
  console.log(`\nQuery: "${query}"`);
  const result = await search(query, { maxResults: 3 });
  
  if (result.error) {
    console.error('❌ Error:', result.error);
  } else {
    console.log(`✅ Found ${result.results.length} results in ${result.queryTime}ms`);
    result.results.forEach((r, i) => {
      console.log(`\n[${i+1}] ${r.title} (${r.scoreLabel} - ${r.score.toFixed(3)})`);
      console.log(`    ${r.url}`);
      console.log(`    ${r.excerpt.slice(0,100)}...`);
    });
  }
}

main().catch(console.error);
```

Run with: `node tests/test-rag.js "your test query"`

---

## VALIDATION CHECKLIST
- [ ] `node tests/test-rag.js "how to deploy"` returns results
- [ ] Results have all required fields (pageId, title, url, excerpt, score, scoreLabel)
- [ ] Error case handled (try with wrong token → should return error object, not crash)
- [ ] queryTime is populated correctly
