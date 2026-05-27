---
name: ieee-search
description: Search IEEE Xplore for academic papers using agent-browser CLI (preferred, headless), Chrome DevTools MCP (non-headless), or Firefox DevTools MCP (fallback) browser automation. No API key required. Supplements paper-search which cannot search IEEE without an API key.
---

# IEEE Xplore Search (Triple MCP)

Search IEEE Xplore via browser automation. Auto-detects available automation and prefers the fastest path.

## Automation Detection

Check availability in this priority order:

1. **agent-browser CLI**: `agent-browser` is available on the system PATH
2. **Chrome DevTools MCP**: `mcp__chrome-devtools__navigate_page` in available tools
3. **Firefox DevTools MCP**: `firefox-devtools` in available tools

Use the first available option. agent-browser runs headless with a custom UA to bypass bot detection.

## agent-browser — Headless Fast Path (Preferred)

Command-line browser automation via `agent-browser` CLI (Rust). Runs headless by default.

**Key tools**: `open`, `snapshot -i`, `click @eN`, `fill @eN`, `wait --text`, `wait --load networkidle`, `eval --stdin`, `network requests`, `tab new`, `tab close`, `close`.

### Why agent-browser first

- **Headless works.** Custom UA avoids IEEE 418 bot detection without needing a headed window.
- **Fast eval.** `eval --stdin` with heredoc runs JS directly in page context.
- **Network inspection.** `network requests` can intercept IEEE's `/rest/search` API responses.
- **Clean per-command model.** No persistent MCP session issues.

### Setup

Ensure agent-browser is installed:
```
npm i -g agent-browser && agent-browser install
```

### Step 1: Open with custom UA and navigate to search

Set a standard browser User-Agent to pass IEEE's bot check. The UA below mimics a recent Chrome on Linux:

```
export IE_UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36"
```

Open the search results page directly:
```
agent-browser open \
  --headers '{"User-Agent": "'"$IE_UA"'"}' \
  "https://ieeexplore.ieee.org/search/searchresult.jsp?queryText=<URL_ENCODED_QUERY>&pageNumber=1"
```

For Chinese/English mixed queries, encode spaces as `+`.

Alternatively, open homepage first, then search:
```
agent-browser open --headers '{"User-Agent": "'"$IE_UA"'"}' "https://ieeexplore.ieee.org/Xplore/home.jsp"
agent-browser wait --text "Search"
agent-browser snapshot -i
# locate the search box (ref @eN) and search button, then:
agent-browser fill @eN "<query>"
agent-browser click @eN
agent-browser wait --load networkidle
```

### Step 2: Wait for results

```
agent-browser wait --text "Showing"
```

### Step 3: Extract paper metadata

**Primary — REST API interception (fastest):**

After the page loads, check network requests for the `/rest/search` call:
```
agent-browser network requests
```

Look for a POST to `/rest/search` with status 200. The response body contains structured JSON with all paper metadata (titles, authors, DOIs, years, abstracts, venues). Parse and present directly.

**Fallback — JavaScript eval DOM extraction:**

```
cat <<'EOF' | agent-browser eval --stdin
const clean = (t) => (t || '').replace(/\s+/g, ' ').trim();
const extractDOI = (text) => {
  const m = text.match(/10\.1109\/[^\s"'>,;\])}]+/);
  return m ? m[0].replace(/[.,;:)\]}>]+$/, '') : '';
};

const containers = document.querySelectorAll('.result-item');
if (containers.length) {
  return Array.from(containers).map(el => {
    const link = el.querySelector('h2 a, h3 a, a[href*="/document/"]');
    return {
      title: link ? clean(link.textContent) : '',
      url: link ? link.href : '',
      authors: clean(el.querySelector('.author, .authors')?.textContent || ''),
      venue: clean(el.querySelector('a[href*="/conference/"], a[href*="/journal/"]')?.textContent || ''),
      year: (el.textContent.match(/\b(19[0-9]{2}|20[0-9]{2})\b/) || [''])[0],
      doi: extractDOI(el.textContent),
      source: 'ieee',
    };
  }).filter(r => r.title);
}

// Fallback: all /document/ links
const seen = new Set();
return Array.from(document.querySelectorAll('a[href*="/document/"]'))
  .filter(a => { const h = a.href; if (seen.has(h)) return false; seen.add(h); return a.textContent.trim().length >= 8; })
  .map(a => ({ title: clean(a.textContent), url: a.href, authors: '', venue: '', year: '', doi: '', source: 'ieee' }));
EOF
```

Returns structured JSON. Sort by year descending, present as a table.

### Step 4: Pagination

Navigate with bumped `pageNumber` in the URL, then wait + extract:
```
agent-browser open --headers '{"User-Agent": "'"$IE_UA"'"}' "https://ieeexplore.ieee.org/search/searchresult.jsp?queryText=<QUERY>&pageNumber=2"
agent-browser wait --text "Showing"
```

### Step 5: Get paper details (abstract, DOI)

Navigate to the paper URL in a new tab:
```
agent-browser tab new "https://ieeexplore.ieee.org/document/<id>/"
agent-browser wait --text "Abstract"
```

Extract with eval:
```
cat <<'EOF' | agent-browser eval --stdin
const clean = (t) => (t || '').replace(/\s+/g, ' ').trim();
let abstract = '';
for (const sel of ['.abstract-text', 'div[class*="abstract"]', 'section[class*="abstract"]']) {
  const el = document.querySelector(sel);
  if (el && el.textContent.trim().length > 50) { abstract = clean(el.textContent); break; }
}
const title = document.querySelector('h1')?.textContent || '';
const authors = Array.from(document.querySelectorAll('.author a, .authors a')).map(a => clean(a.textContent)).join('; ');
const doi = (document.body.textContent.match(/10\.1109\/[^\s"'>,;\])}]+/) || [''])[0]?.replace(/[.,;:)\]}>]+$/, '') || '';
const year = (document.body.textContent.match(/Year:\s*(\d{4})/) || ['',''])[1];
return { title, authors, year, doi, abstract: abstract.substring(0, 2000) };
EOF
```

Close the detail tab after reading:
```
agent-browser tab close 2
```

### Step 6: Present results

```
Found N papers on IEEE Xplore for "<query>":

| # | Title | Authors | Year | DOI |
|---|---|---|---|---|
| 1 | Enhancing Edge AI... | Vishwakarma, V. et al. | 2026 | [10.1109/...](https://doi.org/...) |
```

DOI as clickable markdown link. Sort by year descending.

### Cleanup

```
agent-browser close
```

## Chrome MCP — Non-Headless Path

Use when agent-browser CLI is unavailable but Chrome DevTools MCP is running.

**Important**: IEEE Xplore blocks headless Chrome (Error 418). Chrome MCP must run in **non-headless mode** (`--headed` or without `--headless` flag in MCP config).

Tools: `navigate_page`, `new_page`, `wait_for`, `evaluate_script`, `fill_form`, `click`, `list_network_requests`, `get_network_request`, `take_snapshot`, `screenshot`, `close_page`, `list_pages`, `select_page`.

### Design Principles

- **No DOM polling.** Chrome's `wait_for` is event-driven.
- **REST API interception.** `list_network_requests` + `get_network_request` reads IEEE's `/rest/search` JSON response directly.
- **Batch form filling.** `fill_form` for cookie consent.

### Workflow

1. **Navigate to search URL:**
   ```
   navigate_page("https://ieeexplore.ieee.org/search/searchresult.jsp?queryText=<URL_ENCODED_QUERY>&pageNumber=1")
   ```
   
2. **Handle cookie dialog if present.** Check `take_snapshot` for "Accept All" / "全部接受" buttons, click with `click(uid)`.
   
3. **Wait for results:**
   ```
   wait_for("Showing")
   ```
   
4. **Extract via REST API:** `list_network_requests(resourceTypes=["fetch","xhr"])` to find the `/rest/search` request (POST, status 200), then `get_network_request(reqid=<id>)` to get the structured JSON response. Parse and present.

5. **Fallback DOM extraction** with `evaluate_script` if REST API returns no data.

6. **Paper details:** navigate to document URL, `wait_for("Abstract")`, `evaluate_script` for extraction.

7. **Cleanup:** `close_page(<index>)`

## Firefox MCP — Fallback Path

Use only when both agent-browser and Chrome MCP are unavailable.

- No `wait_for` — use `evaluate_script` DOM polling (every 2-3s, max 8 attempts)
- No `fill_form` — use `fill_by_uid` individually
- No REST API response interception — DOM extraction only
- Firefox requires `--enable-script` flag for `evaluate_script`

## Edge Cases (All Modes)

### Cookie Dialog Blocks Interaction
agent-browser: click the accept button ref from snapshot, or use `find text "Accept All" click`.
Chrome/Firefox: snapshot for button uid + click, or JS button clicker.

### No Results
Suggest broader query. Optionally try paper-search with `-s arxiv,semantic,crossref`.

### Login Wall
If redirected to sign-in (URL contains "signin"), report to user.

### Rate Limiting
Space navigations 1-2s apart. Stop after ~50 results.

### Network request not found (REST API)
If `/rest/search` doesn't appear in network requests, fall back to DOM extraction (eval in agent-browser, evaluate_script in MCP modes).

## Limitations

- PDF access varies by institutional subscription
- No bulk scraping — limit to ~50 results
- DOM-dependent fallback; REST API interception is faster but may break if IEEE changes their API
- agent-browser eval may not capture IEEE's Angular shadow DOM content in some edge cases
