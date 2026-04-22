# Extraction Patterns

Scraping-technique reference. Pagination strategies, large-volume handling, dynamic content, data quality targets. Read `skills/base.md` first for universal rules; recon methodology and validation patterns are in `.claude/agents/scraper.md`.

## Site Archetypes by Difficulty

**Easy:**
- Government databases, public registries
- Static HTML directories
- Sitemap-based sites
- Simple table/list pages

**Moderate:**
- Paginated directories
- Search-result-based sites
- Light JS rendering (data in XHR responses)
- Sites with basic rate limiting

**Hard:**
- E-commerce platforms (dynamic pricing, anti-bot)
- JS-heavy SPAs where data is only in rendered DOM
- Sites with Cloudflare/Akamai protection

**Beyond scope:**
- Login-protected data (no credentials)
- CAPTCHA-gated content
- Sites with explicit ToS prohibition on scraping

## Pagination Patterns

Use the right strategy for the site:

- **Page numbers** (`?page=N`): Increment until empty results. Check for "total pages" indicator.
- **Offset/limit** (`?offset=0&limit=50`): Increment offset by limit each time.
- **Cursor-based** (`?cursor=abc123`): Extract next cursor from response. Stop when cursor is null/missing.
- **Sitemap crawling**: Parse `sitemap.xml` for all URLs, then scrape each. Most reliable when available.
- **Search subdivision**: If a single search returns too many results, break down by letter, region, or category to get complete coverage. Watch for pagination caps. If you hit a round number that looks like a ceiling (e.g., exactly 300 results when more should exist), subdivide the query (by program type, city, etc.) to get full coverage.
- **Radius-search trap**: Some sites do radius-based search, not strict geographic containment. Subdividing by zip code doesn't help because nearby zips return the same overlapping result pool. Instead, subdivide by **category/industry** within a location, or search **distant cities** to avoid overlap. Always deduplicate by unique ID across queries.
- **Multi-query dedup**: When running many queries to bypass pagination caps, collect all results into a single list and deduplicate by the site's unique ID field before output. Don't rely on exact-match dedup alone; some sites return slightly different data for the same record across queries.

## Large Volume Strategies

For jobs with 1,000+ expected records:

- **Checkpoint saves**: Write partial results to disk every 100-500 records. If the scraper crashes at record 4,001, you don't lose the first 4,000. See `toolbox/scraping.md` for format selection (JSON vs JSONL vs staged CSV).
- **Progress logging**: Log current count, estimated remaining, and elapsed time periodically.
- **Resume capability**: Before starting, check if a partial output file exists. If so, determine where to resume from. Deduplicate on resume (checkpoint overlaps are common).
- **Batch processing**: Process in chunks rather than loading everything into memory.
- **Staged execution**: For very large datasets (100K+), break into stages of 50-60 batches (~6 min each). Save to CSV between stages. Avoids execution timeouts.

### Tiered Subdivision for API Result Caps

When APIs cap results per query (e.g., 200 results max), use tiered subdivision:

1. **Level 1**: Broad category (state, metro area, county)
2. **Level 2**: If still capped, narrow by secondary filter (ZIP code, program type, specialty)
3. **Level 3**: If still capped, split further (enumeration type, name prefix a-z)

Document estimated data loss per cap hit for transparency (~50 records per capped query is typical when the cap is 200).

### Full-Text Search Enumeration

When APIs have full-text search with result caps but no structured filters:
- Enumerate via common search terms (top 200 US surnames, 2-letter prefix combinations)
- Deduplicate by unique ID across all search results
- A well-chosen 200-term enumeration can capture the majority of a much larger corpus by exploiting overlap

### Category-Based Subdivision

For radius-based search APIs where geographic subdivision is useless (overlapping results):
- Subdivide by category/industry instead
- Expect 70-85% dedup rate between related categories (HVAC/air-conditioning, accounting/bookkeeping)
- Prioritize diverse categories to maximize unique yield per request

## Dynamic Content Handling

When data loads via JavaScript after initial page render:

- **Wait for specific selectors** -- don't use arbitrary sleep. Use Selenium's `WebDriverWait` with `expected_conditions`.
- **Infinite scroll** -- some sites load results as you scroll. Scroll, check if page height changed, repeat until stable.
- **Iframes** -- if data is in an iframe, switch to it with `driver.switch_to.frame()`. Common on maps embeds and embedded search results.
- **Shadow DOM** -- rare but increasing. Use `driver.execute_script()` to reach into shadow roots.

If Playwright is needed for rendering but the data is large (1,000+ pages), consider: render one page to understand the structure, then find the underlying API call and switch to direct HTTP requests for the bulk work.

## Data Quality Targets

- Field coverage >90% (if a field exists on the page, capture it for 90%+ of records)
- Consistent formatting across all records (don't mix "CA" and "California")
- Source URL captured for every record
- Timestamp of extraction recorded
- Remove true duplicates; use fuzzy matching for near-duplicates when merging sources
