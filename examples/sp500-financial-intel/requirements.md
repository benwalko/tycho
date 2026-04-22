# S&P 500 Financial Intelligence Dataset

## Overview
Build a multi-source enrichment dataset for S&P 500 companies that combines market data, regulatory filings, macro rates, and employment trends into a single deliverable. Zero API signups required.

## Data Sources

### 1. Yahoo Finance (via `yfinance` library, no API key)
Per company:
- Current price, 52-week high/low
- Market cap
- P/E ratio (trailing and forward)
- Dividend yield
- Sector and industry
- Earnings date (next)
- Analyst recommendation mean

### 2. SEC EDGAR (REST API, email in User-Agent, 10 req/sec)
Per company (by CIK lookup):
- Insider transactions last 90 days (Form 4): buys vs sells, share counts, dollar amounts
- Institutional ownership changes from most recent 13F filings (optional if time permits)

### 3. Treasury.gov (direct CSV/JSON, no auth)
- Current Treasury yield curve (1mo, 3mo, 6mo, 1yr, 2yr, 5yr, 10yr, 20yr, 30yr)
- Used as risk-free rate baseline for valuation context

### 4. BLS.gov (public JSON API, no key)
- Employment trends by NAICS sector code
- Map each company's sector to the corresponding BLS employment data
- Latest month-over-month and year-over-year change

## Derived Enrichment Columns
- **Insider Sentiment Score**: net buys minus sells over last 90 days, weighted by dollar size. Normalize to -1 to +1 scale.
- **Earnings Yield vs Treasury Spread**: (1/PE) minus 10-year Treasury yield. Positive = cheap relative to risk-free.
- **Sector Employment Momentum**: YoY employment change in the company's NAICS sector. Positive = growing industry workforce.
- **Composite Signal Flag**: simple combined indicator (e.g., "Bullish" if 2+ of 3 signals positive, "Bearish" if 2+ negative, "Mixed" otherwise)

## Deliverables
- `deliverable.xlsx`: Full dataset, all S&P 500 companies, all columns
- `summary.txt`: Row counts, field completeness percentages, source timestamps, any companies that failed lookup
- `methodology.md`: plain-language explanation of data sources and methodology
- `execution_log.md`: Internal log of approach, milestones, issues encountered

## Technical Notes
- Use `yfinance` for Yahoo data (install via `python -m pip install yfinance`)
- SEC EDGAR: User-Agent must include an email address per SEC policy
- SEC EDGAR rate limit: 10 requests/second max
- S&P 500 ticker list: pull from Wikipedia's "List of S&P 500 companies" table or use a known static source
- BLS sector mapping: GICS sector to NAICS is not 1:1. Use a reasonable best-fit mapping.
- Treasury data: pull the most recent daily yield curve
- If a company fails on one source, still include it with nulls for that source. Don't drop the row.
- Save intermediate results as JSONL checkpoints in case of interruption

## Scope
- All ~500 current S&P 500 companies
- Quality and presentation matter. This is a showcase dataset.
