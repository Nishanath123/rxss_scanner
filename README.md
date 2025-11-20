# rxss_scanner
# VipraTech Reflected XSS Scanner

This single README contains everything: what the project does, how it works (the full logic), how to run it, how to push to GitHub, testing ideas, and notes you can copy into your submission email.
I wrote this in a normal, human style — simple and direct.

---

## What this project is

A small Python tool that checks for **Reflected XSS** by injecting context-aware payloads into given parameters and looking for a marker in the HTTP response. It supports GET and POST, runs requests in parallel, guesses the context of reflections, and writes an HTML report.

This fulfills the assignment requirements:

* PayloadGenerator class with context-specific payloads (including attribute-name).
* Reflection detection using a marker string.
* Supports GET and POST.
* Generates an HTML report and prints a terminal summary.
* Uses threads for parallel scanning and accepts headers/cookies.

---

## Files and structure

```
main.py
scanner/
    engine.py        # scanner orchestration (requests, threading)
    payloads.py      # PayloadGenerator: grouped payloads & marker
    detector.py      # ReflectionDetector: checks for marker + guesses context
    reporter.py      # generates HTML report with Jinja2
templates/
    report.html.j2   # Jinja2 template for the HTML report
requirements.txt
vipratech_report.html  # output (generated)
```
<img width="1893" height="419" alt="image" src="https://github.com/user-attachments/assets/dad76ea8-5583-48a9-ac3c-52f3f380361d" />


---

## Full logic 

Below I explain exactly what each file does and how the pieces tie together.

### `scanner/payloads.py` — PayloadGenerator

Purpose: provide payload strings grouped by likely injection context. Every payload contains a unique marker so the detector can find it later.

Key points:

* `marker`: a fixed string like `VIPRA_XSS_9797`. This is embedded in every payload.
* `self.payloads`: a dict mapping contexts → list of payloads. Supported contexts:

  * `text` — payloads intended for text nodes.
  * `attribute_value` — payloads that break out of attribute values or inject event handlers.
  * `attribute_name` — payloads that attempt to inject into attribute names (this is required by assignment).
  * `js_string` — payloads that break out of JavaScript strings inside `<script>` tags.
  * `tag_name` — payloads that inject tags or tag attributes (e.g., `<img onerror=...>`).
* Methods:

  * `for_context(context)` → returns list for that context.
  * `all_payloads()` → flattens and returns every payload (used now to test everything).
  * `get_marker()` → returns the marker string.

Design notes:

* Grouping by context makes it easy to extend or to test particular contexts only.
* Using the marker rather than raw payload text keeps detection simple and robust.

### `scanner/detector.py` — ReflectionDetector

Purpose: detect whether the marker appears in the response, and try to guess the surrounding context to report a meaningful finding.

Steps it follows:

1. Quick check: `if marker not in html: return not reflected`.
2. If present:

   * Find the position `pos = html.find(marker)`.
   * Create a `snippet` window around `pos` (e.g., 100 chars before and after) so we can inspect surrounding HTML.
   * Parse that snippet with BeautifulSoup to find the node that contains the marker.
3. Guess the context by checking:

   * If the marker is inside a `<script>` node → `js_string`.
   * If the marker appears inside attribute names (parent element's attribute keys include the marker) → `attribute_name`.
   * If the marker appears inside attribute values (attribute values contain the marker) → `attribute_value`.
   * If the payload started with `<` or parsing shows a start of a tag injection → `tag_name`.
   * Otherwise → `text`.
4. Return a dict: `{reflected: True, payload, context, snippet, position}`.

Notes:

* This is heuristic and requests/responses from complex templating engines may confuse it.
* It intentionally keeps detection simple (substring-based) because running a browser JS engine is out of scope.

### `scanner/engine.py` — XSSScanner

Purpose: put everything together — generate payloads, inject them into parameters, send requests, run detection, and save findings.

Key attributes:

* `url`: target URL (without trailing slash).
* `params`: list of parameter names to test (e.g., `["q","search","name"]`).
* `method`: "GET" or "POST".
* `headers`, `cookies`: optional.
* `session`: `requests.Session()` used for all requests.
* `pg`: `PayloadGenerator()` instance.
* `detector`: `ReflectionDetector(pg.get_marker())`.
* `vulnerabilities`: list to accumulate findings.

How `_send_request(param, payload)` works:

* If `GET`:

  * Build `test_url = f"{self.url}?{urlencode({param: payload})}"`.
  * `r = session.get(test_url, timeout=10)`.
* If `POST`:

  * `r = session.post(self.url, data={param: payload}, timeout=10)`.
* Call `detector.is_reflected(r.text, payload)`.
* If reflected, append a dict with keys: `param`, `payload`, `context`, `snippet`, `url`.
* Catch exceptions and skip that request (so one failing request won't stop the whole scan).

How `scan()` works:

* Print starting message.
* `payloads = self.pg.all_payloads()` — for now test all payloads across all params.
* Use `ThreadPoolExecutor(max_workers=15)` to submit `_send_request(param, payload)` for each param × payload pair. This runs requests in parallel for speed.
* After submitting, wait for completion (executor context manager ensures this).
* Call `generate_html_report(self.vulnerabilities, self.url)` and `print_summary(...)`.

Notes:

* You can change thread count or implement throttling to be nice to targets.
* If you want to target specific contexts first, call `pg.for_context(context)` instead of `all_payloads()`.

### `scanner/reporter.py` — report writer

Purpose: render findings to a readable HTML file and print a short terminal summary.

How it works:

* Uses Jinja2 + `templates/report.html.j2`.
* Renders `target`, `timestamp`, `findings`, and `total`.
* Writes `vipratech_report.html` (default) with UTF-8 encoding.
* `print_summary` prints a simple table of findings in terminal.

`templates/report.html.j2` shows each finding in its own block with `Parameter`, `Detected Context`, `Payload`, and `Response Snippet` (snippet uses `| safe` to preserve HTML for visual clarity).

---

## How to run (step-by-step)

1. Clone or copy the repo locally.

2. Create and activate a virtual environment:

   * Windows:

     ```
     python -m venv renv
     venv\Scripts\activate
     ```
  
     ```

3. Install dependencies:

   ```
   pip install -r requirements.txt
   ```

   Example `requirements.txt`:

   ```
   requests
   beautifulsoup4
   lxml
   jinja2
   ```

4. Edit `main.py` to set the URL and parameters you want to test. Example `main.py`:

   ```python
   from scanner.engine import XSSScanner

   if __name__ == "__main__":
       scanner = XSSScanner(
           url="http://testphp.vulnweb.com/search.php",
           parameters=["searchFor", "uname", "query"],
           method="GET"
       )
       scanner.scan()
   ```

5. Run:

   ```
   python main.py
   ```

6. Results:

   * Terminal prints the summary.
   * HTML report written to `vipratech_report.html`. Open it in your browser.

---

## Git commands to push this project

If you want to upload to GitHub:

```bash
git init
git add .
git commit -m "Initial commit: VipraTech Reflected XSS Scanner"
git branch -M main
# Add remote - replace with your repo URL:
git remote add origin git@github.com:<your-username>/vipratech-xss-scanner.git
git push -u origin main
```

Or use HTTPS:

```bash
git remote add origin https://github.com/<your-username>/vipratech-xss-scanner.git
git push -u origin main
```

```
venv/
__pycache__/
*.pyc
vipratech_report.html   # (optional: don't commit report)
```

---

## Testing ideas (quick and simple)

1. Manual run against the test site:

   * `http://testphp.vulnweb.com/search.php` with parameters `["searchFor", "uname", "query"]`.

2. Unit tests suggestions (pytest):

   * Test `PayloadGenerator.get_marker()` is present in every payload returned by `all_payloads()`.
   * Test `ReflectionDetector.is_reflected()` returns `reflected=False` on HTML that does not contain the marker.
   * Create small synthetic HTML strings that include the marker in:

     * `<script>var s = 'VIPRA_XSS_9797'</script>` → expect `js_string`.
     * `<div VIPRA_XSS_9797="x"></div>` → expect `attribute_name`.
     * `<div class="VIPRA_XSS_9797"></div>` → expect `attribute_value`.
     * `<img src=x onerror=alert('VIPRA_XSS_9797')>` → expect `tag_name`.
   * These tests check the heuristics of `_guess_context`.

   ```
   python main.py
   ```

-
