# SKILL 3: PageIndex Retrieval Engine
## src/engines/pageindex_engine.py

### PURPOSE
Implement the runtime search using the pre-built master_index.json.
When a user queries, this engine uses your LLM to reason over the tree structures
and find the most relevant Confluence pages — exactly like a human reading a table of contents.

### HOW PAGEINDEX RETRIEVAL WORKS (Runtime)
```
User query: "how do we handle authentication tokens?"
    ↓
PHASE 1: Page-level scan (fast, broad)
  - Take all page summaries (root node summaries from master_index)
  - Ask LLM: "Which of these pages likely contains info about the query?"
  - LLM returns page_ids ranked by relevance
    ↓
PHASE 2: Tree search on top pages (deep, precise)
  - For each top-ranked page, feed its full tree JSON to the LLM
  - Ask LLM: "Navigate this tree and identify the most relevant node_ids"
  - LLM reasons: "Root mentions API guide → Auth section → Token subsection"
  - Returns specific node_ids
    ↓
PHASE 3: Map nodes → results
  - node_id + page metadata → title, URL, excerpt, score
  - Return standardized result list
```

### KEY DESIGN DECISIONS
- Phase 1 uses ALL page root summaries in a SINGLE LLM call → fast, cheap
- Phase 2 runs per-page tree search IN PARALLEL for top 3-5 pages
- LLM returns JSON arrays → parse cleanly, no regex needed
- Total LLM calls per query: 1 (phase 1) + N (phase 2, N=top pages) ≈ 4-6 calls

---

## PROMPT FOR COPILOT

```
Create src/engines/pageindex_engine.py

### IMPORTS AND SETUP
import json, os, time
from pathlib import Path
from typing import Optional
import sys
sys.path.insert(0, str(Path(__file__).parent.parent.parent))  # project root

import pageindex_llm_patch
pageindex_llm_patch.apply_llm_patch()

import openai

MASTER_INDEX_FILE = Path("data/index/master_index.json")
_master_index = None  # loaded once on first call

### LOAD INDEX (singleton — load once)
def load_index():
    global _master_index
    if _master_index is None:
        if not MASTER_INDEX_FILE.exists():
            raise FileNotFoundError(
                f"Master index not found at {MASTER_INDEX_FILE}. "
                "Run scripts/2_index_all_pages.py and scripts/3_build_search_index.py first."
            )
        _master_index = json.loads(MASTER_INDEX_FILE.read_text())
        print(f"✅ PageIndex loaded: {_master_index['total_pages']} pages, {_master_index['total_nodes']} nodes")
    return _master_index

### PHASE 1: LLM picks top candidate pages
def select_top_pages(query, pages, top_n=5):
    """
    Single LLM call with all page summaries.
    Ask: "Which pages are most relevant to this query?"
    Return top N page_ids.
    
    Prompt:
    You are a document search assistant. Given a user query and a list of Confluence pages 
    (each with title and summary), return the IDs of the {top_n} most relevant pages.
    
    Return ONLY a JSON array of page_id strings, ordered by relevance (most relevant first).
    Example: ["12345", "67890", "11111"]
    
    Query: {query}
    
    Pages:
    {json of [{page_id, title, summary_of_root_node}] for each page}
    
    JSON array of top {top_n} page_ids:
    
    Parse the response: strip whitespace, parse as JSON array.
    On parse error: fall back to first top_n pages (no crash).
    """
    # Build page summaries list
    page_summaries = []
    for page in pages:
        tree = page.get("tree", {})
        root_summary = tree.get("summary", tree.get("title", "No summary available"))
        page_summaries.append({
            "page_id": page["page_id"],
            "title": page["title"],
            "summary": root_summary[:500]  # truncate long summaries
        })
    
    prompt = f"""You are a document search assistant. Given a user query and a list of Confluence wiki pages, return the IDs of the {top_n} most relevant pages.

Return ONLY a JSON array of page_id strings, ordered by relevance (most relevant first).
Example: ["12345", "67890", "11111"]
Do NOT include any explanation, only the JSON array.

Query: {query}

Pages:
{json.dumps(page_summaries, indent=2)}

JSON array of top {top_n} page_ids:"""
    
    client = openai.OpenAI(
        api_key=os.getenv("LLM_API_TOKEN"),
        base_url=os.getenv("LLM_API_BASE_URL")
    )
    
    response = client.chat.completions.create(
        model=os.getenv("LLM_MODEL_NAME"),
        messages=[{"role": "user", "content": prompt}],
        max_tokens=200,
        temperature=0
    )
    
    text = response.choices[0].message.content.strip()
    
    try:
        # Clean up common LLM formatting issues
        text = text.strip().strip("```json").strip("```").strip()
        selected_ids = json.loads(text)
        return selected_ids[:top_n]
    except Exception:
        # Fallback: use first top_n pages
        return [p["page_id"] for p in pages[:top_n]]

### PHASE 2: LLM navigates tree to find relevant nodes
def search_tree_for_query(query, page):
    """
    Given a query and a single page (with its full tree), ask LLM to identify
    the most relevant nodes in that tree.
    
    Returns list of {node_id, relevance_score, reasoning}
    
    Prompt:
    You are navigating a Confluence page tree structure to find content relevant to a query.
    The tree represents a table of contents with summaries for each section.
    
    Identify the 1-3 most relevant node_ids for the query.
    Return a JSON array: [{"node_id": "0003", "score": 0.9, "reason": "directly covers auth tokens"}]
    
    Query: {query}
    Page: {page title}
    Tree: {json of tree, but truncated — only include node_id, title, summary, and nodes recursively}
    
    Parse response → list of {node_id, score, reason}
    On parse error: return [{node_id: root_id, score: 0.5, reason: "fallback"}]
    """
    tree = page.get("tree", {})
    
    # Prune tree for LLM: only send id, title, summary (not full text)
    def prune_tree(node, depth=0):
        if depth > 3: return {"title": node.get("title"), "node_id": node.get("node_id"), "summary": node.get("summary", "")[:200]}
        return {
            "title": node.get("title", ""),
            "node_id": node.get("node_id", ""),
            "summary": node.get("summary", "")[:300],
            "nodes": [prune_tree(c, depth+1) for c in node.get("nodes", [])]
        }
    
    pruned_tree = prune_tree(tree)
    
    prompt = f"""You are navigating a Confluence page tree to find content relevant to this query.
The tree is a hierarchical table-of-contents with LLM-generated summaries.

Return ONLY a JSON array of the 1-3 most relevant nodes.
Format: [{{"node_id": "0001", "score": 0.95, "reason": "directly addresses the query"}}]
Score range: 0.0 (not relevant) to 1.0 (highly relevant).
Do NOT include explanation outside the JSON array.

Query: {query}
Page title: {page['title']}

Tree:
{json.dumps(pruned_tree, indent=2)}

JSON array:"""
    
    client = openai.OpenAI(
        api_key=os.getenv("LLM_API_TOKEN"),
        base_url=os.getenv("LLM_API_BASE_URL")
    )
    
    response = client.chat.completions.create(
        model=os.getenv("LLM_MODEL_NAME"),
        messages=[{"role": "user", "content": prompt}],
        max_tokens=400,
        temperature=0
    )
    
    text = response.choices[0].message.content.strip().strip("```json").strip("```").strip()
    
    try:
        node_results = json.loads(text)
        return node_results if isinstance(node_results, list) else []
    except Exception:
        root_id = tree.get("node_id", "0001")
        return [{"node_id": root_id, "score": 0.5, "reason": "fallback - parse error"}]

### PHASE 3: Map nodes to standardized result format
def resolve_node_to_result(node_id, score, reason, page, index):
    """
    Given a node_id, find its data in all_nodes and build the result dict.
    
    Find matching node from index["all_nodes"] where node["node_id"] == node_id AND node["page_id"] == page_id
    
    Return:
    {
        "page_id": ...,
        "title": "{page_title} > {node_title}",  ← include breadcrumb
        "url": page_url,                          ← Confluence page URL (no deep linking needed)
        "excerpt": node summary,
        "score": score from LLM (0.0-1.0),
        "score_label": "HIGH" if score >= 0.8 else "MEDIUM" if score >= 0.5 else "LOW",
        "node_path": node["node_path"],           ← breadcrumb list
        "reasoning": reason,                      ← LLM's explanation
        "metadata": {"node_id": ..., "node_title": ...}
    }
    
    If node_id not found in all_nodes, use page root data as fallback.
    """

### MAIN SEARCH FUNCTION
def search(query, options=None):
    """
    Main entry point. Returns standardized engine result.
    
    options: {
        max_results: 5,
        top_pages_to_search: 3,    ← how many pages to deep-search
        timeout_seconds: 20
    }
    """
    if options is None:
        options = {}
    
    max_results = options.get("max_results", 5)
    top_pages = options.get("top_pages_to_search", 3)
    start_time = time.time()
    
    try:
        index = load_index()
        pages = index["pages"]
        
        if not pages:
            return {"engine": "pageindex", "results": [], "query_time_ms": 0, "error": "No pages in index"}
        
        # Phase 1: Pick top candidate pages (1 LLM call)
        top_page_ids = select_top_pages(query, pages, top_n=top_pages)
        
        # Phase 2: Deep tree search on top pages (parallel LLM calls)
        top_pages_data = [p for p in pages if p["page_id"] in top_page_ids]
        
        all_node_results = []
        for page in top_pages_data:
            node_hits = search_tree_for_query(query, page)
            for hit in node_hits:
                result = resolve_node_to_result(
                    hit["node_id"], hit["score"], hit.get("reason", ""), page, index
                )
                if result:
                    all_node_results.append(result)
        
        # Sort by score, deduplicate by page_id (keep best scoring per page)
        seen_pages = set()
        final_results = []
        for r in sorted(all_node_results, key=lambda x: x["score"], reverse=True):
            if r["page_id"] not in seen_pages:
                seen_pages.add(r["page_id"])
                final_results.append(r)
            if len(final_results) >= max_results:
                break
        
        query_time_ms = int((time.time() - start_time) * 1000)
        return {
            "engine": "pageindex",
            "results": final_results,
            "query_time_ms": query_time_ms,
            "error": None,
            "metadata": {
                "pages_scanned": len(pages),
                "pages_deep_searched": len(top_pages_data),
                "total_node_hits": len(all_node_results)
            }
        }
    
    except Exception as e:
        return {
            "engine": "pageindex",
            "results": [],
            "query_time_ms": int((time.time() - start_time) * 1000),
            "error": str(e)
        }

### TEST CONNECTION
def test_connection():
    """Verify index is loadable and LLM is reachable."""
    try:
        index = load_index()
        assert index["total_pages"] > 0, "Index has no pages"
        
        # Quick LLM test
        client = openai.OpenAI(
            api_key=os.getenv("LLM_API_TOKEN"),
            base_url=os.getenv("LLM_API_BASE_URL")
        )
        client.chat.completions.create(
            model=os.getenv("LLM_MODEL_NAME"),
            messages=[{"role": "user", "content": "Say OK"}],
            max_tokens=5
        )
        return True
    except Exception as e:
        print(f"Connection test failed: {e}")
        return False
```

---

## TEST FILE PROMPT

```
Create tests/test_pageindex_engine.py

import sys, os
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))
from src.engines.pageindex_engine import search, test_connection

def main():
    query = sys.argv[1] if len(sys.argv) > 1 else "how to authenticate API calls"
    
    print(f"🔗 PageIndex Engine Test")
    print(f"Query: '{query}'\n")
    
    # Test 1: Connection
    ok = test_connection()
    print(f"{'✅' if ok else '❌'} Index + LLM connection: {'OK' if ok else 'FAILED'}\n")
    if not ok:
        return
    
    # Test 2: Search
    result = search(query, {"max_results": 3, "top_pages_to_search": 3})
    
    if result["error"]:
        print(f"❌ Error: {result['error']}")
        return
    
    print(f"✅ Found {len(result['results'])} results in {result['query_time_ms']}ms")
    print(f"   Pages scanned: {result['metadata']['pages_scanned']}")
    print(f"   Pages deep-searched: {result['metadata']['pages_deep_searched']}\n")
    
    for i, r in enumerate(result["results"]):
        print(f"[{i+1}] {r['title']}")
        print(f"     Path: {' > '.join(r['node_path'])}")
        print(f"     Score: {r['score']:.2f} ({r['score_label']})")
        print(f"     Reason: {r.get('reasoning', 'N/A')}")
        print(f"     URL: {r['url']}\n")

if __name__ == "__main__":
    main()
```

Run: `python tests/test_pageindex_engine.py "how do we deploy to production"`

---

## VALIDATION CHECKLIST
- [ ] `python tests/test_pageindex_engine.py "onboarding"` returns results
- [ ] Results include `node_path` showing tree navigation (e.g. `["Guide", "Setup", "Auth"]`)
- [ ] `reasoning` field shows LLM's explanation for picking each node
- [ ] Different queries return different pages (not always the same page)
- [ ] Error case: corrupt master_index.json → returns error dict, doesn't crash server
