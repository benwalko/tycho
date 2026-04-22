# Execution Log - S&P 100 Earnings Season Monitor

## Phase 0 Recon

Probed all three data sources in a single script (`scripts/01_recon.py`).

- **Wikipedia S&P 100**: `pd.read_html` with default urllib UA returned 403. Switched to `requests.get` with a Chrome UA, passed HTML into `pd.read_html` via `StringIO`. Table index 2 is the constituents table with columns `Symbol`, `Name`, `Sector`. 101 rows (Alphabet has both GOOG and GOOGL in the index).
- **yfinance**: `Ticker.info` returns sector/industry/forwardPE/recommendationMean. `Ticker.earnings_dates` returns a 25-row DataFrame with tz-aware "Earnings Date" index and columns "EPS Estimate", "Reported EPS", "Surprise(%)". `Ticker.recommendations` returns a 4-row pivot (0m, -1m, -2m, -3m) with `strongBuy`, `buy`, `hold`, `sell`, `strongSell` counts, not the per-day upgrade/downgrade log I was expecting from older yfinance versions.
- **`^TNX`**: 10Y Treasury yield works. yfinance reports in percent units directly (`Close = 4.286` means 4.286%). No division by 100 needed.
- **SEC `company_tickers.json`**: Works with `User-Agent: sp100-earnings-monitor/1.0 (benjwalko@gmail.com)`. 10,335 entries. Key observation: SEC's `ticker` field uses dashes for share classes (`BRK-B`), not dots. Wikipedia uses dots (`BRK.B`). Had to normalize.

## Approach

Single-pass scrape over 101 constituents:
1. Load S&P 100 from Wikipedia (one request).
2. Load SEC CIK map (one request).
3. Fetch ^TNX once.
4. For each ticker: `yf.Ticker` call pulls `info`, `earnings_dates`, `recommendations`, `revenue_estimate`, `quarterly_income_stmt`, `history` (range around earnings date). Then one `data.sec.gov/submissions/CIK*.json` request for the 10-Q lookup. 0.25s pacing per ticker plus 0.3s before each SEC request. Checkpoints to `scripts/checkpoint.json` every 10 tickers via atomic temp+rename.
5. Build phase reads checkpoint, computes derived columns (beat/miss flags, reactions, earnings yield, alignment), writes `output/deliverable.xlsx` with formatted headers, and generates `summary.txt` and `methodology.md`.

## Pivots

- **Revenue lookup**: Initial code checked only `qis.loc["Total Revenue", closest_column]`. For JPM the 2026-03-31 column had NaN for "Total Revenue" but populated rows for other metrics, and the closest-date logic was matching wrong columns. Switched to: iterate through the 3 nearest period-end dates on or before the earnings date, and within each try "Total Revenue" → "Operating Revenue" → "Net Interest Income" until we find a non-null. Banks and insurers now populate correctly. Final Revenue Actual coverage among reporters: 100%.
- **SEC CIK lookup**: Initial code only tried the Wikipedia-form ticker (`BRK.B`). SEC stores `BRK-B` with a dash. Flipped the lookup order to try yfinance-form first (`yf_ticker = ticker.replace(".", "-")`), then original, then base without suffix.

## Rate Limits Observed

- `data.sec.gov`: 0 × 429s across 101 sequential requests at 0.3s spacing (~3 req/s). Well under the 5 req/s ceiling noted in the site profile.
- `finance.yahoo.com` (via yfinance): No throttling at roughly 3-4 tickers/sec across 101 calls. Consistent with the prior S&P 500 run's observation.
- `en.wikipedia.org`: Default urllib UA blocked (403). Any reasonable browser UA succeeds.

## Discoveries

- **yfinance revenue_estimate has different row labels post-earnings**: When a quarter has just reported, yfinance shifts the `revenue_estimate` row labels. Sometimes `0q` is the just-reported quarter, sometimes it's `-1q`. My code iterates through `-1q` → `0q` → `+1q` and takes the first populated `avg`.
- **"Reported EPS" NaN for same-day earnings**: Six companies with earnings date 2026-04-22 (the window end = today) have an EPS estimate but no actual in `earnings_dates`. These are after-market reports that hadn't posted by scrape time. Kept in the output with EPS Actual blank.
- **BRK-B has no recommendationMean**: Yahoo exposes `recommendationKey = "none"` for Berkshire. No sell-side rollup is published. Analyst Rec Mean coverage is 99% (100/101); this is the single gap.
- **10-Q filing match rate is lower than earnings match rate**: 29 companies reported earnings in the window but only 8 of them had a matching 10-Q filing date in the window. Banks routinely file 10-Qs 10-20 days after the earnings release, pushing the filing date outside a 30-day window that starts at 2026-03-23. Not a bug.

## Validation Results

Quality targets from requirements vs actual:
- Ticker / Company / Sector / Forward P/E: 100% (target 100%). PASS.
- Current Analyst Rec: 99% (BRK.B has none on source). At target.
- EPS Estimate coverage among reporters: 96.6% (28/29). Only GEV (GE Vernova) is missing an estimate; its earnings date is today and yfinance hadn't populated either estimate or actual.
- EPS Actual coverage among reporters: 79.3% (23/29). The 6 gaps are all same-day (2026-04-22) reports not yet posted. In effect 100% of companies whose earnings have actually happened.
- Revenue Estimate / Actual / Surprise among reporters: 100% / 100% / 100%.
- Price On Earnings Close: 100%. Price +1D: 75.9%. Price +5D: 31.0%. The +1D and +5D gaps are strictly structural: if you reported today, there is no +1 trading day yet; if you reported in the last five trading days, there is no +5D yet.
- 10-Q filing coverage: 27.6% of reporters. Discussed above.

## What Didn't Work

- First attempt at `pd.read_html` on Wikipedia returned 403 (default urllib UA). Fixed by fetching via `requests` with a browser UA.
- First revenue lookup only checked "Total Revenue" at the single closest column; returned None for banks when that cell was NaN. Fixed with multi-row fallback.
- First SEC CIK lookup used Wikipedia ticker form with dots; SEC stores dashes. Fixed by checking both forms.

## Recommendations

- `skills/sites/sec-gov-edgar.md` could note that `ticker` in `company_tickers.json` uses **dashes** for share classes. Matches yfinance's convention, differs from Wikipedia's dots.
- For future earnings runs, document the yfinance quirks:
  - `revenue_estimate` row keys shift between `-1q` and `0q` depending on timing relative to the quarter.
  - `recommendations` returns a monthly pivot now, not per-event actions.
  - Actual revenue for banks sometimes requires falling back past "Total Revenue" to "Operating Revenue" or "Net Interest Income" when yfinance's primary field hasn't been populated yet.
- For a tight earnings window ending on the scrape date, expect structural gaps in +1D/+5D price fields. Not worth extending the window just for completeness; instead document it in the summary.

## Files Produced

- `scripts/01_recon.py` - Phase 0 probe of all sources.
- `scripts/02_scrape.py` - Main scrape, writes `scripts/checkpoint.json`.
- `scripts/03_build_output.py` - Builds Excel, summary.txt, methodology.md.
- `scripts/02_smoke.py` - Small smoke-test harness used during development.
- `scripts/debug_revenue_brk.py` - One-off diagnostic script.
- `output/deliverable.xlsx`
- `output/summary.txt`
- `output/methodology.md`
