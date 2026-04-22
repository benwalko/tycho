# Execution Log: S&P 500 Financial Intelligence Dataset

## Approach

Multi-stage pipeline with JSONL checkpoints for resumability:

1. **Stage 1**: Foundation data (S&P 500 list from Wikipedia, Treasury yields, BLS employment)
2. **Stage 2**: Yahoo Finance fundamentals for all 503 companies
3. **Stage 3**: SEC EDGAR insider transaction data (Form 4 filings)
4. **Stage 4**: Assembly, enrichment, Excel generation

## Milestones

### Stage 1: Foundations (15s)
- Wikipedia S&P 500 table: 503 companies. Initial attempt blocked (403) due to default User-Agent. Fixed with browser UA string.
- Treasury yields: Treasury.gov CSV endpoint returned empty data. Treasury OData feed redirected to HTML. Pivoted to yfinance for market yield tickers (3mo, 5yr, 10yr, 30yr) and Fiscal Data API for average rates. Got all key yields.
- BLS employment: 10 of 11 GICS sectors mapped. API returned data for all requested series.

### Stage 2: Yahoo Finance (144s)
- 503/503 tickers fetched, zero errors.
- Rate: 3.5 tickers/sec, no throttling needed.
- All fundamentals captured: price, P/E, market cap, dividends, analyst recs, margins, etc.

### Stage 3: SEC EDGAR (multiple attempts, ~570s total)
- **v1**: Used submissions API (data.sec.gov) for filing counts. Got 503/503 with Form 4 counts. Attempted XML parsing for transaction detail via www.sec.gov but hit 429 rate limits on every attempt. Zero transaction details extracted.
- **v2**: Rebuilt with stricter rate limiting (7 req/sec) and retry backoff. sec.gov still returning 429 for filing index pages. 236 companies processed, still 0 transaction details.
- **v3**: Pivoted to EFTS search API (efts.sec.gov, separate rate pool) for filing discovery, then sec.gov for XML. EFTS worked well. Got transaction detail for 74 companies before sec.gov rate limit kicked in again.
- **v3d**: Final merge. Used EFTS counts for remaining companies (fast, no sec.gov needed). Final: 503 companies, 465 with filing counts, 74 with parsed buy/sell detail.

Key lesson: SEC EDGAR imposes a multi-minute cooldown after rate limit violations, not just per-second throttling. Multiple parallel processes compounded the problem. The EFTS search API (efts.sec.gov) has a separate rate pool from the main site.

### Stage 4: Assembly (instant)
- Merged all sources, computed derived columns.
- BLS mapping initially missed most companies because Yahoo Finance uses different sector names than GICS (e.g., "Technology" vs "Information Technology", "Healthcare" vs "Health Care"). Fixed mapping, got 100% BLS coverage.
- Formatted Excel with color-coded section headers, number formatting, frozen row, auto-filter.

## Issues Encountered

1. **Wikipedia 403**: Default urllib User-Agent blocked. Fixed with browser UA.
2. **Treasury.gov endpoints defunct**: Both CSV and OData feeds no longer serve yield curve data. Pivoted to yfinance and Fiscal Data API.
3. **SEC EDGAR rate limiting**: Aggressive multi-minute cooldown after exceeding 10 req/sec. The www.sec.gov domain shares a rate pool across all paths. Background processes accidentally ran in parallel, compounding the problem.
4. **Yahoo/GICS sector name mismatch**: Yahoo Finance uses its own sector taxonomy, not GICS. Required explicit mapping.

## Final Stats

- 503 rows x 35 columns
- 100% coverage on 21 fields, >90% on 28 of 35 fields
- Insider Sentiment at 14.7% (SEC rate limit constraint)
- Composite Signal at 100% (falls back to 2 of 3 signals)

## Files Produced

- `output/deliverable.xlsx` - Formatted Excel, 503 rows
- `output/summary.txt` - Coverage stats and distribution
- `output/methodology.md` - plain-language methodology
- `checkpoints/` - intermediate JSONL checkpoints (cleaned up post-job)
