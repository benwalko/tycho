---
domain: example.com
aliases: [api.example.com, www.example.com]
last_probed: YYYY-MM-DD
status: green
protection: none
rendering: api
approach: requests
rate_limit_rps: 5
proxy: none
---

# example.com

One-line description of the site and what data it provides.

## Access

- Base URL(s)
- Authentication requirements (API key, User-Agent, cookies, none)
- Header requirements (if any special headers are needed)
- Whether CORS/CSRF matters

## Data

- **Extraction method:** JSON API / HTML parsing / embedded state / other
- **Key endpoints:** specific URLs with example query params
- **Fields available:** list of fields returned per record
- **Record types:** what kind of objects exist (companies, listings, products, etc.)

## Pagination

- **Type:** page numbers / offset+limit / cursor / sitemap / none
- **Page size:** max per request
- **Cap:** maximum total results per query (if any)
- **Strategy:** how to get complete coverage (subdivision, etc.)

## What Doesn't Work

- Approaches that were tried and failed
- Common pitfalls to avoid

## Gotchas

- Quirks that cost time to figure out
- Undocumented requirements
- Non-obvious data structure decisions

## Rate Limits at Scale

Report observed limits at real volume, not just tested:
- "X req/s sustained across Y total requests, zero 429s"
- "Blocked after Z requests at 1 req/s, cooldown of N seconds"

## Job History (optional)

- Date / job name: summary of what was done and how
