# Data Cleaning & Validation

## Phone Number Normalization (US)

```python
import re

def normalize_phone(raw: str) -> str:
    """Normalize US phone number to (XXX) XXX-XXXX format."""
    if not raw:
        return ""
    digits = re.sub(r"\D", "", raw)
    if len(digits) == 11 and digits.startswith("1"):
        digits = digits[1:]
    if len(digits) != 10:
        return raw
    return f"({digits[:3]}) {digits[3:6]}-{digits[6:]}"
```

## Address Standardization

```python
STATE_ABBREVS = {
    "alabama": "AL", "alaska": "AK", "arizona": "AZ", "arkansas": "AR",
    "california": "CA", "colorado": "CO", "connecticut": "CT", "delaware": "DE",
    "florida": "FL", "georgia": "GA", "hawaii": "HI", "idaho": "ID",
    "illinois": "IL", "indiana": "IN", "iowa": "IA", "kansas": "KS",
    "kentucky": "KY", "louisiana": "LA", "maine": "ME", "maryland": "MD",
    "massachusetts": "MA", "michigan": "MI", "minnesota": "MN", "mississippi": "MS",
    "missouri": "MO", "montana": "MT", "nebraska": "NE", "nevada": "NV",
    "new hampshire": "NH", "new jersey": "NJ", "new mexico": "NM", "new york": "NY",
    "north carolina": "NC", "north dakota": "ND", "ohio": "OH", "oklahoma": "OK",
    "oregon": "OR", "pennsylvania": "PA", "rhode island": "RI", "south carolina": "SC",
    "south dakota": "SD", "tennessee": "TN", "texas": "TX", "utah": "UT",
    "vermont": "VT", "virginia": "VA", "washington": "WA", "west virginia": "WV",
    "wisconsin": "WI", "wyoming": "WY", "district of columbia": "DC",
}

def standardize_address(address: str) -> str:
    """Basic address cleanup: trim, normalize state names, fix zip."""
    if not address:
        return ""
    addr = " ".join(address.split())
    for full, abbr in STATE_ABBREVS.items():
        addr = re.sub(rf"\b{full}\b", abbr, addr, flags=re.IGNORECASE)
    addr = re.sub(r"\b(\d{5})-?\d{4}\b", r"\1", addr)
    return addr.strip()
```

## Email Validation

```python
def validate_email(email: str) -> str:
    """Basic email validation and cleanup. Returns cleaned email or empty string."""
    if not email:
        return ""
    email = email.strip().lower()
    pattern = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
    if re.match(pattern, email):
        return email
    return ""
```

## Price and Currency Cleaning

```python
def clean_price(raw: str) -> float | None:
    """Extract numeric price from messy strings like '$1,299.99', 'USD 50', 'Free'."""
    if not raw:
        return None
    raw = raw.strip().lower()
    if raw in ("free", "n/a", "tbd", "call", "contact", "-", ""):
        return None
    cleaned = re.sub(r"[^\d.]", "", raw)
    if not cleaned:
        return None
    try:
        return round(float(cleaned), 2)
    except ValueError:
        return None
```

## URL Normalization

```python
from urllib.parse import urlparse, urlunparse, urljoin

def normalize_url(url: str, base_url: str = "") -> str:
    """Normalize URL: lowercase scheme/host, strip fragments, resolve relative."""
    if not url:
        return ""
    if base_url and not url.startswith("http"):
        url = urljoin(base_url, url)
    parsed = urlparse(url)
    return urlunparse((
        parsed.scheme.lower(),
        parsed.netloc.lower().rstrip("/"),
        parsed.path.rstrip("/") or "/",
        parsed.params,
        parsed.query,
        "",
    ))
```

## Data Validation

```python
def validate_data(data: list[dict]) -> dict:
    """Run basic quality checks on scraped data."""
    if not data:
        return {"row_count": 0, "issues": ["No data"]}
    headers = list(data[0].keys())
    row_count = len(data)
    null_pcts = {}
    for h in headers:
        nulls = sum(1 for row in data if not row.get(h))
        null_pcts[h] = round(nulls / row_count * 100, 1)
    seen = set()
    dupes = 0
    for row in data:
        key = tuple(sorted(row.items()))
        if key in seen:
            dupes += 1
        seen.add(key)
    coverage = {h: round(100 - null_pcts[h], 1) for h in headers}
    return {
        "row_count": row_count,
        "duplicate_rows": dupes,
        "field_coverage": coverage,
        "null_percentages": null_pcts,
        "fields_below_90pct": [h for h, c in coverage.items() if c < 90],
    }
```

## Fuzzy Deduplication

```python
from rapidfuzz import fuzz, process

def fuzzy_dedup(records: list[dict], key_fields: list[str], threshold: int = 85) -> list[dict]:
    """Deduplicate records using fuzzy string matching on key fields."""
    def make_key(r):
        return " | ".join(str(r.get(f, "")).strip().lower() for f in key_fields)
    seen_keys = []
    unique = []
    for record in records:
        key = make_key(record)
        if not key.strip(" |"):
            unique.append(record)
            continue
        match = process.extractOne(key, seen_keys, scorer=fuzz.token_sort_ratio)
        if match and match[1] >= threshold:
            continue
        seen_keys.append(key)
        unique.append(record)
    return unique
```

## Pandas Data Cleaning

```python
import pandas as pd

def clean_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    """Standard cleaning pass for scraped data."""
    for col in df.select_dtypes(include="object").columns:
        df[col] = df[col].str.strip()
    df = df.dropna(how="all")
    df = df.drop_duplicates()
    df = df.reset_index(drop=True)
    return df
```
