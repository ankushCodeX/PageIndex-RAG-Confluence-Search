# SKILL: MAS Plugin Interface — MCP-Compatible Adapter
## Copilot Skill — src/mas-plugin.js + README-MAS.md

### PURPOSE
Make the Confluence AI Search project independently pluggable into your Multi-Agent System (MAS).
Exposes a clean MCP-compatible interface so any MAS agent can call this as a tool.

---

## PROMPT FOR COPILOT — MAS Plugin

```
Create src/mas-plugin.js — MCP-compatible tool definition for MAS integration.

This file serves two purposes:
1. Defines the tool schema (what the MAS agent "sees")
2. Provides a local callable function (for MAS agents running in same Node.js process)

### TOOL DEFINITION (MCP Schema)

const TOOL_DEFINITION = {
  name: "confluence_search",
  description: "Search Confluence knowledge base using dual AI algorithms (RAG vector search + PageIndex link-graph search). Returns ranked pages with relevance scores and a winner verdict comparing both approaches.",
  input_schema: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "Natural language search query or topic to find in Confluence"
      },
      engines: {
        type: "array",
        items: { type: "string", enum: ["rag", "pageindex"] },
        description: "Which search engines to use. Default: both. Use single engine for speed.",
        default: ["rag", "pageindex"]
      },
      maxResults: {
        type: "integer",
        description: "Maximum results per engine (1-10). Default: 5",
        default: 5,
        minimum: 1,
        maximum: 10
      },
      spaceKey: {
        type: "string",
        description: "Confluence space key to search within (e.g. 'ENG', 'HR'). Omit to search all spaces."
      }
    },
    required: ["query"]
  }
};

### CALLABLE FUNCTION (for in-process MAS use)

async function callTool(input) {
  // Validates input against schema
  // Calls orchestrator.search()
  // Returns MCP tool_result format
  
  const { query, engines, maxResults, spaceKey } = input;
  
  // Input validation
  if (!query || typeof query !== 'string') throw new Error('query is required');
  if (query.length < 2) throw new Error('query must be at least 2 characters');
  
  const orchestrator = require('./orchestrator');
  const result = await orchestrator.search(query, {
    engines: engines || ['rag', 'pageindex'],
    maxResults: maxResults || 5,
    spaceKey: spaceKey || null,
    returnFormat: 'mcp'
  });
  
  return result;
}

### HTTP ENDPOINT ADAPTER (for remote MAS use)

Create a route in server.js:
POST /mcp/tools/confluence_search
- Same as /api/search but always returns MCP format
- Validates against TOOL_DEFINITION schema
- Returns standard MCP tool_result

### EXPORTS

module.exports = {
  toolDefinition: TOOL_DEFINITION,
  callTool,
  // MAS registration helper
  register: (masInstance) => {
    masInstance.registerTool(TOOL_DEFINITION, callTool);
    console.log('✅ confluence_search tool registered with MAS');
  }
};
```

---

## PROMPT FOR COPILOT — MAS README

```
Create README-MAS.md:

# Confluence Search — MAS Integration Guide

## Quick Integration

### Option 1: HTTP (Remote Tool)
If your MAS supports HTTP tool calls:
- Tool endpoint: POST http://localhost:3000/mcp/tools/confluence_search
- Schema endpoint: GET http://localhost:3000/mcp/tools (returns all tool definitions)

### Option 2: In-Process (Node.js)
If your MAS is Node.js:
const confluenceSearch = require('./confluence-ai-search/src/mas-plugin');
confluenceSearch.register(yourMASInstance);

### Option 3: Standalone CLI
node src/cli.js --query "deployment guide" --engine rag --format json

## Tool Schema (MCP Format)
[Include the TOOL_DEFINITION JSON]

## Example Usage (from MAS Agent)

User: "Find me documentation about our CI/CD pipeline"
Agent calls: confluence_search({ query: "CI/CD pipeline", engines: ["rag","pageindex"], maxResults: 3 })
Agent receives: { winner: "rag", topResults: [...], confidence: 0.82 }
Agent responds: "I found 3 relevant pages about your CI/CD pipeline. The most relevant is..."

## Response Schema
{
  type: "tool_result",
  tool: "confluence_search",
  content: {
    query: string,
    winner: "rag" | "pageindex" | "tie",
    confidence: number,       // 0.0-1.0
    topResults: [             // Top 3 from winning engine
      {
        pageId: string,
        title: string,
        url: string,
        excerpt: string,
        score: number
      }
    ],
    allResults: { rag: [...], pageindex: [...] },  // Full results from both
    queryTime: number         // milliseconds
  }
}
```

---

## CLI TOOL PROMPT

```
Create src/cli.js — command line interface for standalone testing:

#!/usr/bin/env node
// Usage: node src/cli.js --query "text" [--engine rag|pageindex|both] [--space ENG] [--format json|table] [--max 5]

const args = parseArgs(process.argv.slice(2));
const orchestrator = require('./orchestrator');

async function main() {
  const result = await orchestrator.search(args.query, {
    engines: args.engine === 'both' ? ['rag','pageindex'] : [args.engine || 'rag'],
    maxResults: args.max || 5,
    spaceKey: args.space
  });
  
  if (args.format === 'json') {
    console.log(JSON.stringify(result, null, 2));
  } else {
    // Pretty table format
    printTable(result);
  }
}
```

---

## VALIDATION CHECKLIST
- [ ] `GET /mcp/tools` returns TOOL_DEFINITION JSON
- [ ] `POST /mcp/tools/confluence_search` with `{"query":"test"}` returns MCP format
- [ ] `node src/cli.js --query "onboarding" --format json` works standalone
- [ ] require('./mas-plugin').callTool({query: "test"}) works from Node REPL
- [ ] Tool definition is valid MCP schema (no missing required fields)
