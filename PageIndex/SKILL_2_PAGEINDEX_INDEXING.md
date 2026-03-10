# SKILL 2: PageIndex Indexing — Build Tree Index for All Confluence Pages
## scripts/2_index_all_pages.py + scripts/3_build_search_index.py

### PURPOSE
Run the real PageIndex library (from the cloned repo) on each Confluence .md file to
generate a hierarchical tree structure JSON. Then merge all trees into a single
master_index.json that the retrieval engine searches at runtime.

### CRITICAL: HOW PageIndex MARKDOWN MODE WORKS
PageIndex reads your .md file and uses `#` heading levels to build the tree hierarchy:
- `#` → root node (Level 1)
- `##` → child node (Level 2)  
- `###` → grandchild node (Level 3)
Then it calls your LLM to generate a summary for each node.

For 50-60 Confluence pages, this creates ~50-60 tree JSON files.
Each page = one tree. Your Confluence page = one document.

### ENTERPRISE: REDIRECTING PageIndex's LLM CALLS TO YOUR INTERNAL API
PageIndex uses `openai` Python SDK with env var `CHATGPT_API_KEY`.
Your internal LLM exposes an OpenAI-compatible API.
Solution: Set both the key AND the base_url before running.

---

## PROMPT FOR COPILOT — Step A: LLM Override Setup

```
Create a file called pageindex_llm_patch.py in the project root.

This module must be imported BEFORE any pageindex imports to redirect all LLM calls
from OpenAI's servers to our internal LLM endpoint.

import os
import openai

def apply_llm_patch():
    """
    Redirects PageIndex's OpenAI calls to our internal LLM API.
    PageIndex uses the openai Python SDK internally.
    Since our internal LLM is OpenAI-API-compatible, we just change base_url and api_key.
    """
    internal_url = os.getenv("LLM_API_BASE_URL")   # e.g. http://internal-llm/v1
    internal_key = os.getenv("LLM_API_TOKEN")       # your internal token
    model_name = os.getenv("LLM_MODEL_NAME")        # e.g. "gpt-4-internal"
    
    if not internal_url:
        raise ValueError("LLM_API_BASE_URL is required in .env")
    
    # PageIndex reads CHATGPT_API_KEY from env — match this exactly
    os.environ["CHATGPT_API_KEY"] = internal_key
    
    # Override the openai client globally BEFORE pageindex imports it
    openai.api_key = internal_key
    openai.base_url = internal_url
    
    print(f"✅ LLM patched → {internal_url} (model: {model_name})")
    return model_name

# Call immediately on import
_model = None

def get_model():
    return _model or os.getenv("LLM_MODEL_NAME", "gpt-4")
```

---

## PROMPT FOR COPILOT — Step B: Index All Pages

```
Create scripts/2_index_all_pages.py

This script runs PageIndex on every .md file in data/confluence_pages/
and saves the tree JSON to data/index/trees/

### SETUP (first 10 lines of the script)
import sys, os
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

# MUST import and apply patch BEFORE importing pageindex
import pageindex_llm_patch
pageindex_llm_patch.apply_llm_patch()

# NOW import pageindex (it will use our patched openai client)
from pageindex.page_index_md import md_to_tree    # Markdown processing function
import json
from pathlib import Path
import time

### CONFIG
PAGES_DIR = Path("data/confluence_pages")
TREES_DIR = Path("data/index/trees")
METADATA_FILE = Path("data/index/pages_metadata.json")

### CONFIG OBJECT for PageIndex
# PageIndex uses a config object. Match the fields from pageindex/config.yaml
class PageIndexConfig:
    def __init__(self):
        self.model = pageindex_llm_patch.get_model()
        self.if_add_node_summary = "yes"          # Generate LLM summaries (needed for retrieval)
        self.if_add_node_id = "yes"               # Add unique IDs to nodes
        self.if_add_doc_description = "no"        # Skip doc description (save LLM calls)
        self.if_add_node_text = "no"              # Don't embed full text (keep JSON small)
        self.summary_token_threshold = 200         # Only summarize nodes with >200 tokens
        self.if_thinning = "no"                   # No thinning for small pages
        self.thinning_threshold = 5000

### MAIN INDEXING FUNCTION
def index_page(md_filepath, config, trees_dir):
    """
    Run PageIndex on a single .md file.
    Returns the tree JSON dict or None on error.
    """
    page_name = md_filepath.stem  # e.g. "page_12345"
    output_file = trees_dir / f"{page_name}_structure.json"
    
    # Skip if already indexed (allows resuming interrupted runs)
    if output_file.exists():
        print(f"  ⏭️  Skipping {page_name} (already indexed)")
        return json.loads(output_file.read_text())
    
    try:
        # PageIndex markdown mode: reads headings and calls LLM for summaries
        # md_to_tree(md_path, config) → returns tree dict
        tree = md_to_tree(str(md_filepath), config)
        
        if tree:
            output_file.write_text(json.dumps(tree, indent=2))
            return tree
        else:
            print(f"  ⚠️  Empty tree for {page_name}")
            return None
    except Exception as e:
        print(f"  ❌ Error indexing {page_name}: {e}")
        # Save error marker so we know to skip this page
        error_file = trees_dir / f"{page_name}_error.txt"
        error_file.write_text(str(e))
        return None

def main():
    TREES_DIR.mkdir(parents=True, exist_ok=True)
    config = PageIndexConfig()
    
    md_files = sorted(PAGES_DIR.glob("*.md"))
    print(f"Indexing {len(md_files)} pages with PageIndex...")
    print(f"LLM Model: {config.model}")
    print(f"This will make ~{len(md_files) * 3} LLM API calls (est.)\n")
    
    success = 0
    failed = 0
    
    for i, md_file in enumerate(md_files):
        print(f"[{i+1}/{len(md_files)}] Indexing: {md_file.name}")
        tree = index_page(md_file, config, TREES_DIR)
        if tree:
            success += 1
        else:
            failed += 1
        time.sleep(0.5)  # throttle LLM calls
    
    print(f"\n✅ Done! Success: {success}, Failed: {failed}")
    print(f"Tree files saved to: {TREES_DIR}/")

if __name__ == "__main__":
    main()
```

---

## PROMPT FOR COPILOT — Step C: Build Master Search Index

```
Create scripts/3_build_search_index.py

Merges all per-page tree JSONs + Confluence metadata into a single master_index.json.
This is what the retrieval engine loads at startup.

import json
from pathlib import Path

TREES_DIR = Path("data/index/trees")
METADATA_FILE = Path("data/index/pages_metadata.json")
MASTER_INDEX_FILE = Path("data/index/master_index.json")

def flatten_tree_nodes(tree, page_metadata, parent_path=None):
    """
    Recursively flatten the PageIndex tree into a list of searchable nodes.
    Each node gets enriched with the Confluence page metadata.
    
    Returns list of node dicts:
    {
        "node_id": "0003",
        "page_id": "12345",
        "page_title": "API Integration Guide",
        "page_url": "https://confluence.org/pages/12345",
        "node_title": "Authentication",
        "node_path": ["API Integration Guide", "Authentication"],   ← breadcrumb
        "summary": "This section covers token-based auth...",
        "start_index": 2,
        "end_index": 5,
        "children_count": 3
    }
    """
    if parent_path is None:
        parent_path = []
    
    nodes = []
    current_path = parent_path + [tree.get("title", "")]
    
    node = {
        "node_id": tree.get("node_id", ""),
        "page_id": page_metadata["page_id"],
        "page_title": page_metadata["title"],
        "page_url": page_metadata["url"],
        "page_filename": page_metadata["filename"],
        "node_title": tree.get("title", ""),
        "node_path": current_path,
        "summary": tree.get("summary", ""),
        "start_index": tree.get("start_index", 0),
        "end_index": tree.get("end_index", 0),
        "children_count": len(tree.get("nodes", []))
    }
    nodes.append(node)
    
    # Recurse into children
    for child in tree.get("nodes", []):
        nodes.extend(flatten_tree_nodes(child, page_metadata, current_path))
    
    return nodes

def build_master_index():
    # Load all page metadata
    metadata_list = json.loads(METADATA_FILE.read_text())
    metadata_by_pageid = {m["page_id"]: m for m in metadata_list}
    
    master = {
        "version": "1.0",
        "total_pages": 0,
        "total_nodes": 0,
        "pages": [],       # One entry per Confluence page (top-level tree)
        "all_nodes": []    # Flattened list of ALL nodes across all pages (for fast search)
    }
    
    tree_files = sorted(TREES_DIR.glob("*_structure.json"))
    
    for tree_file in tree_files:
        # Extract page_id from filename: page_12345_structure.json → 12345
        page_name = tree_file.stem.replace("_structure", "")  # page_12345
        page_id = page_name.replace("page_", "")
        
        page_meta = metadata_by_pageid.get(page_id)
        if not page_meta:
            print(f"⚠️  No metadata for {page_name}, skipping")
            continue
        
        tree = json.loads(tree_file.read_text())
        
        # Add page-level entry
        master["pages"].append({
            "page_id": page_id,
            "title": page_meta["title"],
            "url": page_meta["url"],
            "tree": tree,         # Full tree (used during retrieval reasoning)
            "ancestor_titles": page_meta.get("ancestor_titles", [])
        })
        
        # Add flattened nodes
        flat_nodes = flatten_tree_nodes(tree, page_meta)
        master["all_nodes"].extend(flat_nodes)
    
    master["total_pages"] = len(master["pages"])
    master["total_nodes"] = len(master["all_nodes"])
    
    MASTER_INDEX_FILE.write_text(json.dumps(master, indent=2))
    print(f"✅ Master index built:")
    print(f"   Pages indexed: {master['total_pages']}")
    print(f"   Total nodes: {master['total_nodes']}")
    print(f"   Saved to: {MASTER_INDEX_FILE}")

if __name__ == "__main__":
    build_master_index()
```

---

## VALIDATION CHECKLIST

After step 2: `python scripts/2_index_all_pages.py`
- [ ] `data/index/trees/` contains ~50-60 `*_structure.json` files
- [ ] Open one JSON — it should have `title`, `node_id`, `summary`, `nodes` fields
- [ ] If a page has no headings (flat HTML), it will produce a single-node tree — that's OK

After step 3: `python scripts/3_build_search_index.py`
- [ ] `data/index/master_index.json` exists
- [ ] `total_pages` matches your Confluence page count
- [ ] `all_nodes` has significantly more entries than pages (proves tree structure worked)
- [ ] `python -c "import json; d=json.load(open('data/index/master_index.json')); print(d['total_pages'], d['total_nodes'])"`

## TROUBLESHOOTING

### "md_to_tree not found" error
The function name may differ across PageIndex versions. Check the actual module:
`grep -r "def.*tree\|def.*md" pageindex/page_index_md.py | head -20`
Then update the import in the script.

### LLM API 401/403 errors during indexing
Your patch isn't working. Add debug print right after patch:
`print(openai.base_url, openai.api_key[:10])`

### tiktoken not on Nexus
Replace token counting with: `def count_tokens(text): return int(len(text.split()) * 1.3)`
Override in pageindex/utils or wherever it's called.

### Pages with no Markdown headings produce flat trees
That's expected. Confluence pages without `## Headers` will produce a single root node.
They still work — the whole page summary becomes the searchable unit.
