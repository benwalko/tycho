# S&P 500 Financial Intelligence Dataset

## What This Is

A financial dataset covering all 503 S&P 500 companies, pulling from four public data sources and computing derived signals. No API keys or paid subscriptions required.

## Data Sources

**Yahoo Finance** (via yfinance library)
Pulled fundamentals for every company: price, P/E ratios, market cap, dividend yield, analyst recommendations, revenue, margins, beta. 100% coverage across 503 tickers.

**SEC EDGAR** (REST API)
Queried Form 4 insider transaction filings for the last 90 days. 465 of 503 companies had insider filing activity. Parsed transaction details (buys vs. sells, dollar amounts) for 74 companies from the actual XML filings. Filing counts are available for all 465.

**Treasury.gov Yield Curve**
Current Treasury yields pulled via Yahoo Finance market data: 3-month (3.61%), 5-year (3.92%), 10-year (4.31%), 30-year (4.94%). Supplemented with Fiscal Data API averages for Bills, Notes, and Bonds.

**BLS Employment Data** (public JSON API)
Sector-level employment trends mapped from GICS sectors to BLS CES series. Covers 10 of 11 GICS sectors. Includes year-over-year and month-over-month employment changes.

## Derived Enrichment Columns

**Insider Sentiment Score** (-1 to +1): Net insider buys minus sells, weighted by dollar value, normalized. Available for 74 companies with parsed Form 4 transaction detail. The rest show filing count only.

**Earnings Yield vs. Treasury Spread**: (1/PE) minus 10-year Treasury yield. Positive means the stock's earnings yield exceeds the risk-free rate. 94% coverage (companies with negative or zero P/E excluded).

**Sector Employment Momentum**: Year-over-year employment change in the company's sector. Positive means the sector is adding jobs. 100% coverage.

**Composite Signal**: Simple indicator combining the three signals above. "Bullish" if 2+ signals positive, "Bearish" if 2+ negative, "Mixed" otherwise. Falls back to 2 signals when insider sentiment is unavailable.

## Coverage Notes

Most fields are at 100% or near-100% coverage. A few gaps:

- Dividend Yield (81%): not all S&P 500 companies pay dividends.
- Insider Sentiment (15%): SEC rate limits prevented parsing transaction XMLs for all 503 companies. Filing counts are complete.
- P/E Trailing (94%): companies with negative earnings don't have a meaningful P/E.

## Technical Details

- S&P 500 list sourced from Wikipedia's current index table.
- SEC EDGAR rate limit: 10 req/sec enforced. User-Agent includes contact email per EDGAR policy.
- BLS sector mapping uses best-fit GICS-to-NAICS translation. Not a perfect 1:1 match, but covers the major sectors.
- Treasury yields are market-close values, not intraday.
- All data fetched on April 16, 2026.
