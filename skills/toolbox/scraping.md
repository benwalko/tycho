# Scraping Patterns

## Rate-Limited Requests Session

```python
import time
import random
import requests

class RateLimitedSession:
    def __init__(self, min_delay=1.0, max_delay=3.0, max_retries=3):
        self.session = requests.Session()
        self.session.headers.update({
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.9",
            "Accept-Encoding": "gzip, deflate, br",
        })
        self.min_delay = min_delay
        self.max_delay = max_delay
        self.max_retries = max_retries
        self._last_request = 0

    def get(self, url, **kwargs):
        elapsed = time.time() - self._last_request
        delay = random.uniform(self.min_delay, self.max_delay)
        if elapsed < delay:
            time.sleep(delay - elapsed)
        for attempt in range(self.max_retries):
            self._last_request = time.time()
            resp = self.session.get(url, **kwargs)
            if resp.status_code == 429:
                backoff = (5 * (2 ** attempt)) + random.uniform(0, 1)
                print(f"429 on {url}, backing off {backoff:.1f}s (attempt {attempt+1})")
                time.sleep(backoff)
                continue
            resp.raise_for_status()
            return resp
        raise Exception(f"Max retries exceeded for {url}")
```

## curl_cffi with TLS Fingerprint Impersonation

```python
from curl_cffi import requests as cffi_requests

def get_with_tls_impersonation(url, **kwargs):
    """Bypass basic bot detection via TLS fingerprint matching."""
    resp = cffi_requests.get(
        url,
        impersonate="chrome131",
        headers={
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.9",
        },
        **kwargs,
    )
    resp.raise_for_status()
    return resp
```

## Form Submission with CSRF Token

```python
from bs4 import BeautifulSoup
from urllib.parse import urljoin

def submit_search_form(session, form_url, search_params: dict) -> str:
    """Handle forms that require a CSRF token or hidden fields."""
    resp = session.get(form_url)
    soup = BeautifulSoup(resp.text, "lxml")
    form = soup.find("form")
    payload = {}
    if form:
        for inp in form.find_all("input", {"type": "hidden"}):
            payload[inp.get("name", "")] = inp.get("value", "")
    payload.update(search_params)
    action = form.get("action", form_url) if form else form_url
    if not action.startswith("http"):
        action = urljoin(form_url, action)
    resp = session.post(action, data=payload)
    resp.raise_for_status()
    return resp.text
```

### ASP.NET ViewState / EventValidation Forms

ASP.NET Web Forms (.aspx) require ALL hidden form fields in every POST, not just visible ones. Missing any one causes a 500 error.

```python
def extract_aspnet_form_data(html: str) -> dict:
    """Extract all ASP.NET hidden form fields including ViewState and EventValidation."""
    soup = BeautifulSoup(html, "lxml")
    payload = {}
    for inp in soup.find_all("input"):
        name = inp.get("name", "")
        if name:
            payload[name] = inp.get("value", "")
    # Must include: __VIEWSTATE, __VIEWSTATEGENERATOR, __EVENTVALIDATION,
    # __EVENTTARGET, __EVENTARGUMENT, __LASTFOCUS (even if empty)
    return payload
```

**Key rules:**
- Extract fresh ViewState/EventValidation from EACH page response. Tokens are page-specific.
- Do NOT construct pagination URLs manually. ASP.NET uses encrypted `q` parameters for pagination. Extract pagination links from `a.linkToPage` elements.
- Include ALL form fields in POST, including empty hidden fields.

## Parallel Array Extraction

Some APIs return data as parallel arrays instead of objects. Must zip by index.

```python
# Example: SEC EDGAR submissions returns parallel arrays:
# {"form": ["10-K", "10-Q", ...], "filingDate": ["2024-01-15", ...], ...}
def zip_parallel_arrays(data: dict, keys: list[str]) -> list[dict]:
    """Zip parallel arrays from API response into list of dicts."""
    arrays = [data.get(k, []) for k in keys]
    return [dict(zip(keys, row)) for row in zip(*arrays)]
```

## Resumable Scraping with Checkpoints

```python
import json
import os

class CheckpointScraper:
    """Save progress periodically so crashes don't lose data."""

    def __init__(self, checkpoint_path: str):
        self.checkpoint_path = checkpoint_path
        self.results = []
        self.completed_urls = set()
        self._load()

    def _load(self):
        if os.path.exists(self.checkpoint_path):
            with open(self.checkpoint_path) as f:
                data = json.load(f)
            self.results = data.get("results", [])
            self.completed_urls = set(data.get("completed_urls", []))
            print(f"Resumed: {len(self.results)} records, {len(self.completed_urls)} URLs done")

    def save(self):
        """Atomic write: write to temp file, then rename. Prevents corruption from concurrent reads."""
        tmp = self.checkpoint_path + ".tmp"
        with open(tmp, "w") as f:
            json.dump({
                "results": self.results,
                "completed_urls": list(self.completed_urls),
            }, f)
        os.replace(tmp, self.checkpoint_path)

    def is_done(self, url: str) -> bool:
        return url in self.completed_urls

    def add(self, url: str, records: list[dict]):
        self.results.extend(records)
        self.completed_urls.add(url)
        if len(self.completed_urls) % 50 == 0:
            self.save()
```

### Checkpoint Format Selection

Choose the right format based on dataset size:

- **JSON** (`CheckpointScraper` above): Good for <10MB. Full document access, easy to inspect. But full rewrites become slow past ~37MB and risk corruption on concurrent access.
- **JSONL** (append-only): Best for large growing datasets (10K+ records). Each line is one JSON record. No I/O bottleneck since you only append, never rewrite.
- **Staged CSV**: Best for multi-stage pipelines with intermediate processing. Run N batches per stage, save to CSV between stages. Useful for avoiding execution timeouts (e.g., 60 batches per stage, ~6 min each).

```python
# JSONL append pattern
def append_jsonl(path: str, records: list[dict]):
    with open(path, "a") as f:
        for r in records:
            f.write(json.dumps(r) + "\n")
```

### Concurrent Agent Coordination

When spawning multiple background agents on the same job:
- Each agent MUST write to its own checkpoint file (indexed by batch_id)
- Use atomic writes (temp file + `os.replace()`) to prevent truncation
- After all agents complete, merge checkpoints in a dedicated synchronous step
- NEVER have multiple processes write to the same checkpoint file. On some OSes, `json.dump()` truncates before writing, so a concurrent read during write returns empty/truncated data.

## Embedded State Object Extraction

Many modern sites (Next.js, React, etc.) embed full data in `<script>` tags as JSON. Common patterns:

```python
import re, json

def extract_preloaded_state(html: str) -> dict | None:
    """Extract window.__PRELOADED_STATE__, __NEXT_DATA__, or similar embedded JSON."""
    patterns = [
        r'window\.__PRELOADED_STATE__\s*=\s*({.*?});\s*</script>',
        r'<script\s+id="__NEXT_DATA__"\s+type="application/json">(.*?)</script>',
        r'window\.__INITIAL_STATE__\s*=\s*({.*?});\s*</script>',
        r'window\.__data\s*=\s*({.*?});\s*</script>',
    ]
    for pattern in patterns:
        match = re.search(pattern, html, re.DOTALL)
        if match:
            try:
                return json.loads(match.group(1))
            except json.JSONDecodeError:
                continue
    return None


def find_data_in_state(state: dict, target_key: str, path: str = "") -> list:
    """Recursively search a state object for a key. Returns list of (path, value) tuples.

    RECON ONLY. Lists are sampled at the first 5 items to bound recursion depth on large
    state blobs. Do NOT use this for extraction; once you locate a path, index into the
    full state directly.
    """
    results = []
    if isinstance(state, dict):
        for k, v in state.items():
            if k == target_key:
                results.append((f"{path}.{k}", v))
            results.extend(find_data_in_state(v, target_key, f"{path}.{k}"))
    elif isinstance(state, list):
        for i, item in enumerate(state[:5]):
            results.extend(find_data_in_state(item, target_key, f"{path}[{i}]"))
    return results
```

**Tip:** When probing a new site, use `find_data_in_state()` with a known field name (like "businessName" or "price") to locate the data path. Then use that path to index into the full state for extraction. The function samples lists at the first 5 items; it will find the path but won't return all values.

## Cloudflare Cookie Session

When Cloudflare is present (check for `cf-ray` header), use a persistent session to maintain the `__cf_bm` cookie:

```python
import requests

def get_cf_session() -> requests.Session:
    """Session that persists Cloudflare cookies across requests."""
    session = requests.Session()
    session.headers.update({
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "en-US,en;q=0.9",
        "Accept-Encoding": "gzip, deflate, br",
    })
    return session
```

The session automatically carries `__cf_bm` and other Cloudflare cookies from the first request to subsequent ones. Do NOT create a new session per request.

## Schema Validation

Define a Pydantic model for your output. Validate every record during extraction, not after.

```python
from pydantic import BaseModel, field_validator
from typing import Optional

class BusinessRecord(BaseModel):
    name: str
    address: str
    city: str
    state: str
    zip_code: str
    phone: Optional[str] = None

    @field_validator('state')
    @classmethod
    def validate_state(cls, v):
        if len(v) != 2:
            raise ValueError(f'Invalid state: {v}')
        return v.upper()

# Usage: validate during extraction, log failures
valid_records = []
failed_records = []

for raw in raw_data:
    try:
        record = BusinessRecord(**raw)
        valid_records.append(record.model_dump())
    except Exception as e:
        failed_records.append({"data": raw, "error": str(e)})

fail_rate = len(failed_records) / max(len(raw_data), 1)
print(f"Validation: {len(valid_records)} passed, {len(failed_records)} failed ({fail_rate:.1%})")
if fail_rate > 0.05:
    print("WARNING: >5% failure rate, investigate before continuing")
```

Adapt the model fields to match each job's output schema. The pattern stays the same.

## Circuit Breaker

Prevents IP bans by pausing after consecutive failures. Add to any loop that makes requests.

```python
import time

consecutive_failures = 0
MAX_FAILURES = 5
COOLDOWN = 30

for url in urls:
    try:
        result = scrape(url)
        consecutive_failures = 0
    except Exception as e:
        consecutive_failures += 1
        print(f"Failure {consecutive_failures}/{MAX_FAILURES}: {e}")
        if consecutive_failures >= MAX_FAILURES:
            print(f"Circuit breaker tripped, cooling down {COOLDOWN}s")
            time.sleep(COOLDOWN)
            consecutive_failures = 0
```

Combine with `RateLimitedSession` for full protection: rate limiting handles normal pacing, circuit breaker handles cascading failures.

## Browser Automation

See `skills/selenium.md` for the full Selenium skill.
