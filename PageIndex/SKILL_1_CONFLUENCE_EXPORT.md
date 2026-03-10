# SKILL 1: Export Confluence Pages → Markdown Files
## scripts/1_export_confluence.py

### PURPOSE
Pull your 50-60 Confluence pages via the Confluence REST API and save each one as a
clean Markdown file (.md) in data/confluence_pages/. This is the input data pipeline
that feeds PageIndex in the next step.

### WHY MARKDOWN?
PageIndex supports two input modes: PDF and Markdown.
- PDF requires `pymupdf` and `PyPDF2` — extra dependencies, potential Nexus risk
- Markdown only needs what's already in requirements.txt
- Confluence HTML → Markdown is a clean, dependency-light conversion
- Use `--md_path` flag with run_pageindex.py → NO PDF libraries needed at all

---

## PROMPT FOR COPILOT

```
Create scripts/1_export_confluence.py

This script exports Confluence pages to Markdown files for PageIndex processing.

### DEPENDENCIES (standard library + requests only)
import os, json, re, time, html
import requests
from pathlib import Path

NO extra dependencies. Use only Python standard library + requests.

### CONFIG (read from .env or env vars)
CONFLUENCE_BASE_URL = os.getenv("CONFLUENCE_BASE_URL")  # e.g. https://yourorg.atlassian.net/wiki
CONFLUENCE_USER = os.getenv("CONFLUENCE_USER")           # email
CONFLUENCE_TOKEN = os.getenv("CONFLUENCE_TOKEN")         # API token
CONFLUENCE_SPACE_KEY = os.getenv("CONFLUENCE_SPACE_KEY") # e.g. "ENG"
OUTPUT_DIR = Path("data/confluence_pages")
METADATA_FILE = Path("data/index/pages_metadata.json")

### STEP 1: Fetch all pages in the space
def fetch_all_pages(space_key):
    """
    GET /wiki/rest/api/content
    params: spaceKey=ENG, type=page, limit=50, start=0, expand=body.storage,version,ancestors
    Use Basic auth: base64(user:token)
    Paginate using the 'next' link in response until no more pages.
    Return list of all page dicts.
    """
    auth = requests.auth.HTTPBasicAuth(CONFLUENCE_USER, CONFLUENCE_TOKEN)
    pages = []
    url = f"{CONFLUENCE_BASE_URL}/rest/api/content"
    params = {
        "spaceKey": space_key,
        "type": "page",
        "limit": 50,
        "start": 0,
        "expand": "body.storage,version,ancestors,metadata.labels"
    }
    while url:
        resp = requests.get(url, params=params, auth=auth)
        resp.raise_for_status()
        data = resp.json()
        pages.extend(data["results"])
        # Pagination: check for _links.next
        next_link = data.get("_links", {}).get("next")
        if next_link:
            url = CONFLUENCE_BASE_URL + next_link
            params = {}  # params are already in the next URL
        else:
            url = None
        time.sleep(0.2)  # rate limit courtesy
    return pages

### STEP 2: Convert Confluence storage format (HTML) to clean Markdown
def confluence_html_to_markdown(html_content, page_title):
    """
    Convert Confluence's storage format HTML to clean Markdown.
    Rules:
    1. Start with "# {page_title}" as the H1 heading
    2. Convert <h1>-<h6> to # ## ### #### ##### ######
    3. Convert <p> to paragraph text (add blank line before and after)
    4. Convert <ul><li> to - bullet
    5. Convert <ol><li> to 1. numbered list
    6. Convert <strong><b> to **text**
    7. Convert <em><i> to *text*
    8. Convert <code> to `code`
    9. Convert <pre><code> blocks to ```\ncode\n```
    10. Convert <a href="URL">text</a> to [text](URL)
    11. Strip all remaining HTML tags with regex
    12. Unescape HTML entities (&amp; &lt; &gt; etc.)
    13. Collapse multiple blank lines to max 2
    14. Strip Confluence-specific macros: <ac:structured-macro>, <ac:parameter>, etc.
    
    Use only re module for this - no html parsing libraries.
    Return clean markdown string.
    """

### STEP 3: Save page as .md file
def save_page_as_markdown(page, output_dir):
    """
    - Filename: page_{page_id}.md  (use page ID, not title, to avoid special chars)
    - First line MUST be: # {page title}
    - Then the converted markdown content
    - Return metadata dict: {page_id, title, url, filename, last_modified, ancestor_titles}
    """
    page_id = page["id"]
    title = page["title"]
    html_content = page["body"]["storage"]["value"]
    
    md_content = confluence_html_to_markdown(html_content, title)
    
    filename = f"page_{page_id}.md"
    filepath = output_dir / filename
    filepath.write_text(md_content, encoding="utf-8")
    
    # Build Confluence URL
    url = f"{CONFLUENCE_BASE_URL}/pages/viewpage.action?pageId={page_id}"
    
    # Extract ancestor titles for breadcrumb context
    ancestors = [a["title"] for a in page.get("ancestors", [])]
    
    return {
        "page_id": page_id,
        "title": title,
        "url": url,
        "filename": filename,
        "filepath": str(filepath),
        "last_modified": page["version"]["when"],
        "ancestor_titles": ancestors,
        "space_key": CONFLUENCE_SPACE_KEY
    }

### MAIN
def main():
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
    Path("data/index").mkdir(parents=True, exist_ok=True)
    
    print(f"Fetching pages from Confluence space: {CONFLUENCE_SPACE_KEY}")
    pages = fetch_all_pages(CONFLUENCE_SPACE_KEY)
    print(f"Found {len(pages)} pages")
    
    metadata_list = []
    for i, page in enumerate(pages):
        print(f"[{i+1}/{len(pages)}] Exporting: {page['title']}")
        meta = save_page_as_markdown(page, OUTPUT_DIR)
        metadata_list.append(meta)
    
    # Save metadata for later use (maps page_id → URL, title, etc.)
    METADATA_FILE.write_text(json.dumps(metadata_list, indent=2))
    print(f"\n✅ Exported {len(metadata_list)} pages to {OUTPUT_DIR}/")
    print(f"✅ Metadata saved to {METADATA_FILE}")

if __name__ == "__main__":
    main()
```

---

## VALIDATION CHECKLIST
After running: `python scripts/1_export_confluence.py`

- [ ] `data/confluence_pages/` contains ~50-60 `.md` files
- [ ] Each .md file starts with `# Page Title` (required for PageIndex heading detection)
- [ ] `data/index/pages_metadata.json` exists and contains all page IDs + URLs
- [ ] Open a sample `.md` file — it should be readable plain text, not HTML soup
- [ ] Check: `wc -l data/confluence_pages/*.md` — each file should have >5 lines

## TROUBLESHOOTING
- 401 error: Wrong token format. Try `Authorization: Bearer TOKEN` instead of Basic
- 403 error: Token doesn't have space read permission
- HTML entities still showing: Add `html.unescape()` call after stripping tags
- Macro content missing: Confluence macros like {code} blocks won't export cleanly — that's OK for 50-60 pages
