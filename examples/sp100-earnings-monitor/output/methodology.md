# Methodology

## Objective
Produce a single-sheet Excel dataset describing the most recent earnings event for each S&P 100 company over the 30 calendar days ending 2026-04-22, enriched with filings metadata, analyst rating trends, and macro rate context.

## Universe
S&P 100 constituents pulled from Wikipedia's S&P 100 page. 101 rows are present because two share classes of Alphabet (GOOG/GOOGL) are both included in the index. Every company is kept even when it did not report earnings in the window, with blank earnings fields and a note.

## Data Sources
- **Yahoo Finance** via the `yfinance` Python library. Used for company metadata (sector, industry, market cap, forward P/E), earnings history and estimates (`Ticker.earnings_dates`), analyst recommendation aggregates (`Ticker.recommendations`), revenue estimates (`Ticker.revenue_estimate`), actual revenue (`Ticker.quarterly_income_stmt`), and daily closing prices (`Ticker.history`). Yahoo's sector taxonomy differs from GICS (for example, "Technology" vs. "Information Technology"). We kept the Wikipedia / S&P 100 sector as the canonical sector and included Yahoo's industry label as a secondary field.
- **SEC EDGAR** via the REST JSON endpoints. `company_tickers.json` provides CIK mapping. `data.sec.gov/submissions/CIK{10-digit}.json` provides filing indexes. We looked for 10-Q filings with a filing date inside the window and recorded accession number and filing URL. No filing content was fetched.
- **10Y Treasury Yield** from Yahoo's `^TNX` ticker (the Treasury's public CSV/OData endpoints were defunct at last check). Most recent close used for every row. 2026-04-22 close = quoted in summary.txt.

## Window
Earnings date or filing date between 2026-03-23 and 2026-04-22 inclusive. Earnings-date comparisons use timezone-aware timestamps (Yahoo reports earnings times with exchange offsets); filings are compared in UTC.

## Derived Columns
- EPS / Revenue Beat-Miss flag: "Beat" if surprise > +1%, "Miss" if surprise < -1%, "In-Line" otherwise.
- Double Beat: both EPS and Revenue flagged Beat.
- 1-Day Reaction (%): close the trading day after earnings divided by close on or before the earnings date.
- 1-Week Reaction (%): close the fifth trading day after earnings, same base.
- Reaction vs Surprise: "Aligned Positive" when both surprise and reaction are positive, "Aligned Negative" when both are negative, "Contra" otherwise.
- Earnings Yield (%): 1 / forward_pe * 100, skipped when forward_pe is null or non-positive.
- Earnings Yield vs 10Y (bps): earnings_yield_percent minus treasury_10y_percent, then multiplied by 100 to report in basis points.
- Rec Trend: compares current month analyst aggregates to one month prior (difference in buy count, sell count, and a weighted score).

## Rate Limits Applied
- SEC data.sec.gov: 0.3s between requests (~3 req/s), well within the 5 req/s ceiling noted in the site profile.
- yfinance: no explicit throttle beyond a 0.25s per-ticker pace; library handles its own pacing.
- Wikipedia: single request.

## Caveats
- Some companies that reported in the window still lack a Revenue Actual value because Yahoo's quarterly income statement had not yet populated the most recent quarter. This is a source-data lag, not a data-quality issue in the pipeline. Those rows still carry EPS actual, EPS surprise, price reactions, and revenue estimate.
- 10-Q filings do not always fall inside the earnings window. Some banks, insurers, and calendar-year reporters file their 10-Q several days after the earnings release, which can put the filing outside a tight 30-day window.
- Analyst recommendation trend is derived from a monthly rollup (0m, -1m) rather than a per-event upgrade/downgrade log, which Yahoo no longer exposes publicly.
- Yahoo's sector taxonomy maps imperfectly to GICS. The S&P 100 / GICS sector label is retained as canonical.
- Forward P/E can be negative when consensus earnings are expected to be negative; Earnings Yield is null in those cases.
