# IDOT contract data scraper

A single-page browser tool that scrapes Illinois Department of Transportation (IDOT) contract letting pages and exports structured CSV data.

## What it does

IDOT publishes contract award data across individual HTML detail pages, one per contract. This tool accepts a batch of contract URLs (pasted or uploaded as a `.txt` file), fetches each page, parses the HTML for contractor names, award prices, counties, section numbers, and federal project numbers, and writes everything to a downloadable CSV.

A second version (`new.html`) extends the workflow to two steps: it starts from a letting notice index page, filters contracts by county and award status, then automatically follows each contract's detail link to scrape the full data — no URL list required.

### Output fields

| Field | Source |
|---|---|
| `Letting_Date` | Contract header |
| `Contract_ID` | Contract header |
| `Contractor_Number` | Award badge / contractor info block |
| `Contractor_Name` | Award badge / contractor info block |
| `Award_Price` | Award badge |
| `County` | Info table |
| `Section` | Info table |
| `Federal_Project_No` | Info table |
| `Source_URL` | Input |

---

## Technical problems solved

This tool required solving five distinct problems, each of which blocked progress until resolved. They are documented here in roughly the order they were encountered.

---

### 1. Python vs. JavaScript for HTML parsing

**The problem.** IDOT's contract pages have irregular, nested HTML with no clean API or consistent class naming. The award winner is wrapped in a `<span class="alert-success">` inside a `<li>` inside an unsorted list; contact data spreads across sibling table rows with no stable IDs. JavaScript's `querySelector` and `querySelectorAll` can navigate this, but BeautifulSoup's `find_parent()`, `find_next_sibling()`, and CSS selector chaining are meaningfully better for this kind of traversal. Rewriting working Colab notebook logic in JavaScript introduced fragility at every step.

**The solution.** Run Python directly in the browser using [Pyodide](https://pyodide.org) — a WebAssembly build of CPython. The scraping logic from the Colab prototype was ported to the browser nearly intact. BeautifulSoup4 is installed at runtime via `micropip`. The Python functions execute in the browser tab, with no server involved.

**The tradeoff.** Pyodide's full build is approximately 10MB. Initialization takes 5–15 seconds on first load depending on connection speed. The tool shows a loading badge and keeps the action button disabled until the runtime is ready.

---

### 2. CORS blocking browser fetch requests to the IDOT server

**The problem.** IDOT's government server (`webapps1.dot.illinois.gov`) does not include permissive CORS headers. A direct `fetch()` call from a browser tab to that domain is blocked by the browser's same-origin policy before the request even leaves the machine. This is not a server configuration the tool can control.

**The solution.** All target URLs are routed through [corsproxy.io](https://corsproxy.io), a public CORS proxy that fetches the target server-side and returns the content with permissive `Access-Control-Allow-Origin` headers. The proxy URL is constructed by URL-encoding the full target URL and appending it as a query parameter:

```python
import urllib.parse
encoded = urllib.parse.quote(target_url, safe='')
proxy_url = f"https://corsproxy.io/?{encoded}"
```

The URL-encoding step is not optional. IDOT contract URLs contain query parameters (e.g., `?id=12345`). Without encoding, those parameters are parsed as part of the proxy's own query string rather than the target URL, producing 404 errors or wrong pages.

**The dependency risk.** `corsproxy.io` is a free third-party service. If it goes down or begins rate-limiting, the tool breaks. A Google Apps Script web app deployed as a personal proxy (`UrlFetchApp.fetch(url)`, returned with `Access-Control-Allow-Origin: *`) is a more durable alternative for production use.

---

### 3. Python's `requests` library doesn't work in the browser sandbox

**The problem.** Pyodide runs CPython in WebAssembly inside a browser tab. The browser sandbox does not expose raw sockets. Python's standard `requests` library relies on sockets and fails silently or throws cryptic errors when called from inside Pyodide — even though the code looks correct.

There are two working approaches, and the repo uses both across different versions of the tool.

**Solution A: `pyodide-http` patch (used in `index.html`).** Install the `pyodide-http` package via `micropip` and call `pyodide_http.patch_all()` before any requests are made. This monkey-patches the `requests` library to route calls through the browser's `fetch()` API instead of sockets. After patching, `requests.get(url)` works as expected.

```python
import micropip
await micropip.install('pyodide-http')
import pyodide_http
pyodide_http.patch_all()
import requests  # now browser-compatible
```

**Solution B: `pyfetch` (used in `new.html`).** Pyodide ships a native async HTTP function, `pyodide.http.pyfetch`, that wraps the browser's `fetch()` API directly without requiring patching. It is the cleaner approach when starting fresh rather than porting existing `requests`-based code.

```python
from pyodide.http import pyfetch
response = await pyfetch(url, headers={...})
html = await response.text()
```

`pyfetch` is async-native. `requests` with `pyodide-http` is synchronous and slightly simpler to read, making it easier to port Colab notebook code with minimal changes.

---

### 4. UI freezing during the scrape loop

**The problem.** Pyodide's async Python runs on the same JavaScript event loop as the browser UI. A tight async loop — fetch, parse, fetch, parse — holds the event loop continuously. The progress bar and log panel stop updating; the browser tab appears frozen. The scrape is running, but the user has no feedback.

**The solution.** Insert `await asyncio.sleep(0.05)` at each iteration of the loop, before or after the fetch. This yields control back to the JavaScript event loop long enough for the DOM to repaint and process any pending UI updates.

```python
import asyncio

for i, url in enumerate(urls):
    js_callback(i + 1, len(urls), url, "FETCHING")
    await asyncio.sleep(0.05)  # yield to JS event loop
    response = requests.get(proxy_url)
    # ...
```

`asyncio.sleep(0)` technically yields but is not reliable enough for consistent repaints. `0.05` seconds (50ms) is sufficient.

---

### 5. Passing JavaScript functions into Python as callbacks

**The problem.** The progress bar and log panel are DOM elements controlled by JavaScript. Python, running inside Pyodide, needs to call back into JavaScript to report progress after each URL — but there is no obvious mechanism for this in a single-file HTML tool.

**The solution.** Pyodide allows JavaScript functions to be passed directly as arguments to Python callables. Pyodide converts them to Python-callable proxy objects automatically.

```javascript
// JS side: retrieve the Python function and call it with a JS callback
const pyFunction = pyodide.globals.get('process_urls');
const result = await pyFunction(urlList, progressCallback);

function progressCallback(current, total, url, status) {
  updateProgressBar(current, total);
  addLogEntry(url, status);
}
```

```python
# Python side: call the JS callback like any Python function
async def process_urls(urls_list, js_callback):
    for i, url in enumerate(urls_list):
        js_callback(i + 1, len(urls_list), url, "FETCHING")
        await asyncio.sleep(0.05)
        # ... fetch and parse
        js_callback(i + 1, len(urls_list), url, "SUCCESS")
```

An earlier version injected the callback via `pyodide.globals.set('progressCallback', fn)` and then called the Python function through a dynamically constructed `runPythonAsync` string. The direct-argument pattern above is cleaner and avoids string-constructing Python code in JavaScript.

---

### 6. Parsing IDOT's irregular HTML structure

**The problem.** IDOT contract pages do not use a consistent, machine-friendly structure. The awarded contractor's name and price appear inside a `<span class="alert-success">` badge embedded in an unordered list, not in a labeled table row. Extracting the contractor number and name requires splitting a single string on the first space. The info table (county, section, federal project number) has `<th>` headers but no IDs, so lookups depend on finding the header by text content and then navigating to the sibling `<td>`.

**The solution.** BeautifulSoup's relational traversal methods handle this cleanly. The winner logic:

```python
winner_badge = soup.find("span", class_="alert-success")
parent_li = winner_badge.find_parent("li")
contractor_name_tag = parent_li.find("strong")
```

The table lookup uses an "anchored" pattern — find the header cell by string content, determine its column index, then find the corresponding data cell in the first `tbody` row:

```python
def get_table_value_by_header(header_name):
    header = soup.find("th", string=lambda t: t and header_name in t)
    headers = header.find_parent("tr").find_all("th")
    col_index = headers.index(header)
    table = header.find_parent("table")
    first_row_cells = table.find("tbody").find("tr").find_all("td")
    return first_row_cells[col_index].get_text(" ", strip=True)
```

Post-processing splits the raw contractor field (`"12345 ACME CONSTRUCTION INC"`) into `Contractor_Number` and `Contractor_Name` on the first space boundary.

---

## File inventory

| File | Description |
|---|---|
| `index.html` | Current production tool. Accepts a list of contract URLs, scrapes each, exports CSV. Uses `requests` + `pyodide-http` patch + `corsproxy.io`. |
| `new.html` | Extended version. Starts from a letting notice index URL, filters by county and status, auto-follows detail links. Uses `pyfetch` natively. |
| `cl_IDOT_scraper.html` | Early version. Pyodide + BeautifulSoup working, but missing `pyodide-http` patch and CORS proxy. Functional in some local environments; fails on GitHub Pages. |
| `idotscrapernew2.html` | Intermediate iteration. |
| `new2.html` | Intermediate iteration. |

---

## Dependencies

All loaded from CDN at runtime — no install required.

| Library | Version | Purpose |
|---|---|---|
| Pyodide | 0.24.1 / 0.25.0 | Python runtime in WebAssembly |
| BeautifulSoup4 | latest via micropip | HTML parsing |
| pyodide-http | latest via micropip | Patches `requests` for browser compat (`index.html`) |
| requests | latest via micropip | HTTP client (`index.html`) |
| pyfetch | built into Pyodide | Async HTTP client (`new.html`) |

---

## Known limitations

**Proxy dependency.** `corsproxy.io` is a public free service. Downtime or rate limits will break the tool. A Google Apps Script web app is a more reliable self-hosted alternative.

**Slow startup.** Pyodide initializes in 5–15 seconds. This is inherent to loading the WebAssembly runtime and cannot be significantly reduced without switching to a different architecture.

**IDOT page structure.** The HTML parsing logic is keyed to specific class names and structural patterns in IDOT's current page templates. If IDOT redesigns their contract pages, the selectors will need to be updated.

**Single-threaded.** URLs are scraped sequentially, one at a time. Large batches (50+ URLs) take proportionally longer.
