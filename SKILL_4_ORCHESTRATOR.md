# SKILL: Search Orchestrator + Winner Judge
## Copilot Skill — src/orchestrator.js + src/judge.js

### PURPOSE
Run both engines IN PARALLEL, collect results, and determine a winner.
This is the core of the "dual engine battle" feature.

---

## PROMPT FOR COPILOT — orchestrator.js

```
Create src/orchestrator.js — runs RAG and PageIndex engines in parallel.

const rag = require('./engines/rag');
const pageindex = require('./engines/pageindex');
const judge = require('./judge');

async function search(query, options = {}) {
  /**
   * Options:
   *   engines: ['rag', 'pageindex'] — which engines to run (default: both)
   *   maxResults: 5 — results per engine
   *   spaceKey: null — optional Confluence space filter
   *   returnFormat: 'json' | 'mcp' — output format (default: json)
   */
  const {
    engines = ['rag', 'pageindex'],
    maxResults = 5,
    spaceKey = null,
    returnFormat = 'json'
  } = options;

  const startTime = Date.now();
  const engineOptions = { maxResults, spaceKey };

  // Run selected engines in parallel (Promise.allSettled so one failure doesn't kill both)
  const tasks = [];
  if (engines.includes('rag')) tasks.push(rag.search(query, engineOptions));
  if (engines.includes('pageindex')) tasks.push(pageindex.search(query, engineOptions));

  const settled = await Promise.allSettled(tasks);

  // Collect results (handle both fulfilled and rejected)
  const results = {};
  let taskIndex = 0;
  if (engines.includes('rag')) {
    const r = settled[taskIndex++];
    results.rag = r.status === 'fulfilled' ? r.value : { engine:'rag', results:[], error: r.reason?.message, queryTime: 0 };
  }
  if (engines.includes('pageindex')) {
    const r = settled[taskIndex++];
    results.pageindex = r.status === 'fulfilled' ? r.value : { engine:'pageindex', results:[], error: r.reason?.message, queryTime: 0 };
  }

  // Judge picks winner
  const verdict = judge.evaluate(results, query);

  const response = {
    query,
    totalTime: Date.now() - startTime,
    results,
    verdict,
    metadata: {
      enginesRun: engines,
      spaceKey,
      timestamp: new Date().toISOString()
    }
  };

  // MCP format for MAS integration
  if (returnFormat === 'mcp') {
    return toMCPFormat(response);
  }
  return response;
}

function toMCPFormat(response) {
  // Standard MCP tool result format for plugging into MAS
  return {
    type: 'tool_result',
    tool: 'confluence_search',
    content: {
      query: response.query,
      winner: response.verdict.winner,
      confidence: response.verdict.confidence,
      topResults: response.verdict.winner === 'tie'
        ? [...(response.results.rag?.results || []).slice(0,3)]
        : (response.results[response.verdict.winner]?.results || []).slice(0,3),
      allResults: response.results,
      queryTime: response.totalTime
    }
  };
}

module.exports = { search };
```

---

## PROMPT FOR COPILOT — judge.js

```
Create src/judge.js — evaluates which engine performed better.

Scoring criteria (weighted):
1. Result count (20%): Engine that found MORE results scores higher
2. Average score (40%): Engine with higher average relevance scores wins
3. Top score (25%): Engine with the highest single result score wins  
4. Speed (15%): Faster engine gets a bonus (inverted: lower time = higher score)

function evaluate(results, query) {
  /**
   * Returns:
   * {
   *   winner: 'rag' | 'pageindex' | 'tie',
   *   confidence: 0.0-1.0,        // How decisive was the win
   *   scores: { rag: X, pageindex: Y },  // Weighted scores 0-100
   *   explanation: string,         // Human-readable verdict
   *   breakdown: { ... }           // Per-criteria scores for UI visualization
   * }
   */
  
  const engineIds = Object.keys(results).filter(k => !results[k].error);
  if (engineIds.length === 0) return { winner: 'none', confidence: 0, explanation: 'Both engines failed', scores: {}, breakdown: {} };
  if (engineIds.length === 1) return { winner: engineIds[0], confidence: 1.0, explanation: `Only ${engineIds[0]} returned results`, scores: {}, breakdown: {} };

  const scores = {};
  const breakdown = {};

  for (const engineId of engineIds) {
    const engine = results[engineId];
    const engineResults = engine.results || [];
    
    // Criterion 1: Result count (normalized against max possible = maxResults)
    const countScore = Math.min(engineResults.length / 5, 1.0) * 20;
    
    // Criterion 2: Average relevance score
    const avgScore = engineResults.length > 0
      ? engineResults.reduce((sum, r) => sum + r.score, 0) / engineResults.length
      : 0;
    const avgScoreWeighted = avgScore * 40;
    
    // Criterion 3: Top result score
    const topScore = engineResults[0]?.score || 0;
    const topScoreWeighted = topScore * 25;
    
    // Criterion 4: Speed (max 1000ms is the benchmark; faster = better)
    const speedScore = Math.max(0, 1 - (engine.queryTime / 10000)) * 15;
    
    const total = countScore + avgScoreWeighted + topScoreWeighted + speedScore;
    scores[engineId] = Math.round(total);
    breakdown[engineId] = {
      resultCount: Math.round(countScore),
      avgRelevance: Math.round(avgScoreWeighted),
      topRelevance: Math.round(topScoreWeighted),
      speed: Math.round(speedScore),
      queryTime: engine.queryTime
    };
  }

  // Determine winner
  const ragScore = scores.rag || 0;
  const pageIndexScore = scores.pageindex || 0;
  const diff = Math.abs(ragScore - pageIndexScore);
  
  let winner, confidence, explanation;
  
  if (diff < 5) {
    winner = 'tie';
    confidence = 0.5;
    explanation = `Both engines performed similarly (RAG: ${ragScore}, PageIndex: ${pageIndexScore}). The query was ambiguous enough that both approaches found relevant content.`;
  } else if (ragScore > pageIndexScore) {
    winner = 'rag';
    confidence = Math.min(diff / 30, 1.0);
    explanation = `Vector RAG wins by ${diff} points. Semantic similarity search excelled at understanding the conceptual intent of this query.`;
  } else {
    winner = 'pageindex';
    confidence = Math.min(diff / 30, 1.0);
    explanation = `PageIndex wins by ${diff} points. The keyword + graph traversal approach found more precisely relevant pages for this query.`;
  }

  return { winner, confidence, scores, breakdown, explanation };
}

module.exports = { evaluate };
```

---

## API ROUTE PROMPT (add to server.js)

```
Add this route to server.js:

app.post('/api/search', async (req, res) => {
  const { query, engines, maxResults, spaceKey, returnFormat } = req.body;
  
  if (!query || query.trim().length < 2) {
    return res.status(400).json({ error: 'Query must be at least 2 characters' });
  }
  
  const orchestrator = require('./src/orchestrator');
  const result = await orchestrator.search(query.trim(), {
    engines: engines || ['rag', 'pageindex'],
    maxResults: Math.min(maxResults || 5, 10),
    spaceKey: spaceKey || process.env.CONFLUENCE_DEFAULT_SPACE || null,
    returnFormat: returnFormat || 'json'
  });
  
  res.json(result);
});

app.get('/api/spaces', async (req, res) => {
  // Return list of Confluence spaces the user has access to
  const axios = require('axios');
  const config = require('./src/config');
  const auth = Buffer.from(`${config.CONFLUENCE_USER}:${config.CONFLUENCE_TOKEN}`).toString('base64');
  
  const response = await axios.get(`${config.CONFLUENCE_BASE_URL}/rest/api/space`, {
    headers: { Authorization: `Basic ${auth}` },
    params: { limit: 50, type: 'global' }
  });
  
  res.json(response.data.results.map(s => ({ key: s.key, name: s.name })));
});
```

---

## ORCHESTRATOR TEST PROMPT

```
Create tests/test-orchestrator.js:

const orchestrator = require('../src/orchestrator');

async function main() {
  const query = process.argv[2] || 'how to onboard new developers';
  console.log(`\n🎯 Dual Engine Battle: "${query}"\n`);
  console.log('Running RAG + PageIndex in parallel...\n');
  
  const result = await orchestrator.search(query, { maxResults: 3 });
  
  console.log(`⏱️  Total time: ${result.totalTime}ms`);
  console.log(`\n📊 VERDICT: ${result.verdict.winner.toUpperCase()} WINS (confidence: ${(result.verdict.confidence * 100).toFixed(0)}%)`);
  console.log(`   ${result.verdict.explanation}\n`);
  
  console.log('📈 Scores:');
  Object.entries(result.verdict.scores).forEach(([engine, score]) => {
    const bar = '█'.repeat(Math.round(score / 5)) + '░'.repeat(20 - Math.round(score / 5));
    console.log(`   ${engine.padEnd(12)} [${bar}] ${score}/100`);
  });
  
  console.log('\n--- RAG Results ---');
  (result.results.rag?.results || []).forEach((r, i) =>
    console.log(`[${i+1}] ${r.title} (${r.score.toFixed(3)})`));
  
  console.log('\n--- PageIndex Results ---');
  (result.results.pageindex?.results || []).forEach((r, i) =>
    console.log(`[${i+1}] ${r.title} (${r.score.toFixed(3)})`));
}

main().catch(console.error);
```

---

## VALIDATION CHECKLIST
- [ ] `node tests/test-orchestrator.js "API design"` runs both engines in parallel
- [ ] Verdict includes winner, scores, and explanation
- [ ] If one engine fails, the other still returns results
- [ ] MCP format output works: add `--mcp` flag or test via curl with `returnFormat: "mcp"`
- [ ] `curl -X POST localhost:3000/api/search -H "Content-Type:application/json" -d '{"query":"deployment"}'` returns full dual result
