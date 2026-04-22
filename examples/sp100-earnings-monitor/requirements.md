# S&P 100 Earnings Season Monitor

## Overview

Build a multi-source dataset tracking the most recent earnings release for every S&P 100 company over the past 30 calendar days. Combine fundamentals, filings, analyst activity, and rate context into a single enriched Excel deliverable.

Zero API signups required. All sources are public.

## Scope

- Universe: S&P 100 companies (~100 tickers)
- Time window: earnings released between 2026-03-23 and 2026-04-22 (last 30 days)
- If a company did not report in this window, include the row with blank earnings fields and a note. Do not drop it.

## Data Sources

### 1. Yahoo Finance (via `yfinance` library, no API key)
Per company:
- Ticker, company name, sector, industry
- Most recent earnings date (must fall in the window; otherwise mark as no-report-this-window)
- EPS estimate (consensus pre-earnings)
- EPS actual (as reported)
- EPS surprise ($ and %)
- Revenue estimate
- Revenue actual
- Revenue surprise ($ and %)
- Price on earnings date close
- Price 1 trading day after earnings
- Price 5 trading days after earnings (1-week reaction)
- Forward P/E
- Current analyst recommendation (mean rating)
- Analyst recommendation trend (how many upgrades vs downgrades in the past 30 days, if available)

### 2. SEC EDGAR (REST API)
Per company:
- CIK from `company_tickers.json`
- Most recent 10-Q filing date in the window (if one was filed around earnings)
- Accession number and filing URL

See the existing site profile at `skills/sites/sec-gov-edgar.md` for endpoints, rate limits, and UA requirements.

### 3. Treasury Yield Context
- Current 10-year Treasury yield (market close, most recent)
- Use `yfinance` ticker `^TNX` (confirmed working during the prior S&P 500 run). The treasurydirect.gov CSV and OData endpoints were defunct at last check; do not waste time on them.

## Derived Enrichment Columns

- **1-Day Reaction %**: (price_1d_after - price_earnings_close) / price_earnings_close * 100
- **1-Week Reaction %**: (price_5d_after - price_earnings_close) / price_earnings_close * 100
- **EPS Beat/Miss Flag**: "Beat" if EPS surprise > 0, "Miss" if < 0, "In-Line" if within +/- 1%
- **Revenue Beat/Miss Flag**: same logic on revenue surprise
- **Double Beat**: true if both EPS and Revenue beat
- **Earnings Yield %**: 1 / forward_pe * 100 (skip when forward_pe is null or negative)
- **Earnings Yield vs 10Y Spread**: earnings_yield_% minus treasury_10y_%
- **Reaction-vs-Surprise Alignment**: "Aligned Positive" (beat + positive price reaction), "Aligned Negative" (miss + negative reaction), "Contra" when surprise and price reaction disagree

## Deliverables

All in `output/`:
- `deliverable.xlsx`: one row per S&P 100 company, formatted headers, frozen top row, auto-width columns
- `summary.txt`: total companies, companies with earnings in window, beat rate, miss rate, double-beat count, field coverage percentages, data-source timestamps
- `methodology.md`: plain-language explanation of sources, methodology, and any caveats
- `execution_log.md`: internal log: approach, pivots, discoveries, rate limits observed, recommendations

## Technical Notes

- Install yfinance: `python3 -m pip install yfinance pandas openpyxl pydantic requests`
- SEC EDGAR requires a contact email in the User-Agent header. Format: `"YourTool/1.0 (contact@example.com)"`
- yfinance `Ticker.earnings_dates` returns upcoming + historical earnings. Filter to the 30-day window.
- yfinance `Ticker.recommendations` returns a DataFrame of analyst actions with dates; count upgrades vs downgrades in the window.
- Save intermediate progress as JSONL checkpoints per the framework's Resumability guidance
- Put all scripts in `jobs/sp100-earnings-monitor/scripts/` (the orchestrator's post-job retrospective reads them)

## Scope Guardrails

- Do not fetch intraday prices. Daily closes are sufficient.
- Do not parse 10-Q filing content. Metadata only (date, accession number, URL).
- Companies that didn't report in the window: include the row with a note, don't drop.
- Target completion time: under 30 minutes.

## Quality Targets

- 100% coverage on ticker, company, sector, forward P/E, current analyst rec
- 90%+ coverage on earnings-related fields *among companies that reported in the window*
- Treasury 10Y yield: single value, same for all rows
- No fabricated data. If a field is empty on the source, it's empty in the output.
