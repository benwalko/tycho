---
domain: sec.gov
aliases: [efts.sec.gov, www.sec.gov, data.sec.gov]
last_probed: 2026-04-22
status: green
protection: basic
rendering: api
approach: requests
rate_limit_rps: 5
proxy: none
---

# sec.gov (EDGAR)

The US Securities and Exchange Commission's EDGAR system. Public company filings, insider transactions, institutional holdings. One of the cleanest public data APIs available.

## Access

- EDGAR full-text search API at `efts.sec.gov/LATEST/search-index`
- Submissions data at `data.sec.gov/submissions/CIK{10-digit-padded}.json`
- **Requires email in User-Agent header.** SEC blocks requests without a contact email. Use: `"User-Agent": "YourTool/1.0 (contact@example.com)"`
- Main sec.gov pages have Akamai WAF and are more aggressive with rate limits than the API endpoints.
- No API key required.

## Data

- **Extraction method:** JSON API
- **Company tickers endpoint:** `www.sec.gov/files/company_tickers.json` (all public companies, ~8K)
- **Filing search:** `efts.sec.gov/LATEST/search-index?q=*&forms=10-K,10-Q&...`
- **Submissions endpoint:** `data.sec.gov/submissions/CIK0000320193.json` (example for AAPL)
- **Fields:** companyName, CIK, ticker, filingDate, formType, filingUrl, accessionNumber
- Company tickers file: CIK, ticker, title

## Pagination

- **Type:** offset/limit via `from` and `size` params
- **Page size:** up to 200 per request (`size=200`)
- **Cap:** API returns up to 10,000 results per query. For larger sets, subdivide by date range.
- Date range params: `startdt`, `enddt` (format: YYYY-MM-DD)

## What Doesn't Work

- Default Python urllib User-Agent (403 from Akamai on main pages)
- Requesting without email in UA header
- `size` > 200 (API ignores, returns 200 max)

## Gotchas

- The `company_tickers.json` file is a single static download, not paginated.
- **Share-class ticker format:** `company_tickers.json` stores share classes with **dashes**, not dots. `BRK-B`, not `BRK.B`. This matches yfinance's convention but differs from Wikipedia's S&P lists. When joining across sources, try both forms.
- Filing search `from` param can go up to 10,000. Beyond that, must subdivide by date range.
- www.sec.gov imposes a MULTI-MINUTE cooldown after rate limit violations, not just per-second throttling. Once 429'd, wait several minutes before resuming.
- The EFTS search API (efts.sec.gov) has a separate rate pool from www.sec.gov. If hit by 429s on www.sec.gov, pivot to EFTS for filing discovery.
- Submissions endpoint returns parallel arrays (form[], filingDate[], etc.), not objects. Zip by index.
- SIC codes missing for ~13% of companies (SPACs, newer registrants, foreign filers).
- Responses are ~100-200KB each. For large runs, use JSONL append-only format, not full JSON rewrites.
- Form 4 (insider transactions) requires XML parsing of the actual filing documents, not just the index. XML parsing hits www.sec.gov rate pool.

## Rate Limits at Scale

- **efts.sec.gov:** 5 req/s sustained across 10,000+ requests works reliably. Zero 429s observed at this rate.
- **www.sec.gov:** More aggressive. 3-4 req/s is safer. Multi-minute cooldowns trigger easily if exceeded.
- **data.sec.gov:** Similar to efts, around 5 req/s is fine.

## Strategy Notes

For large-scale filing extraction:
1. Get full ticker/CIK list from `company_tickers.json` (one request)
2. Hit `data.sec.gov/submissions/...` for each CIK to get filing index
3. Only fetch individual filing XML if the job requires transaction-level detail
4. Use JSONL append-only for large result sets
5. Separate the "filing metadata" phase from the "filing content" phase; they have different rate characteristics
