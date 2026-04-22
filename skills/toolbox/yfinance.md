# yfinance Patterns

Library-specific quirks and patterns for the `yfinance` Python package. yfinance wraps Yahoo Finance's undocumented-but-stable endpoints. It's the most reliable way to pull equity market data without an API key.

## Install

```bash
python3 -m pip install yfinance pandas
```

## Throughput

At S&P 100 / S&P 500 scale: ~3-4 tickers/second sustained, zero throttling needed. No explicit rate limiting required at the library level. Observed across multiple 100-500 ticker runs.

## Treasury Yields (no need for Treasury.gov)

Treasury.gov's public CSV and OData yield-curve endpoints have been defunct for multiple runs. Use yfinance tickers instead:

| Ticker | Maturity |
|--------|----------|
| `^IRX` | 13-week T-bill |
| `^FVX` | 5-year |
| `^TNX` | 10-year |
| `^TYX` | 30-year |

yfinance reports these in percent directly. `Close = 4.286` means 4.286%. No division by 100.

```python
import yfinance as yf
tnx = yf.Ticker("^TNX").history(period="5d")["Close"].iloc[-1]
```

## Earnings Data (`Ticker.earnings_dates`)

Returns a DataFrame indexed by timezone-aware `Earnings Date` with columns:
- `EPS Estimate`
- `Reported EPS`
- `Surprise(%)`

Includes both upcoming and historical earnings (typically last 4 quarters + next 4). Filter on the date index to isolate a window.

**Gotcha, same-day reports:** Companies reporting after market close on the scrape date show an `EPS Estimate` but null `Reported EPS`. Not a bug; the actual hasn't posted yet. Handle as null, don't drop the row.

## Revenue Estimates (`Ticker.revenue_estimate`)

Returns a small DataFrame with rows keyed by relative quarter (`-1q`, `0q`, `+1q`, `+2q`) and columns for `avg`, `low`, `high`, `numberOfAnalysts`.

**Gotcha, row keys shift after earnings:** Immediately after a quarter is reported, the row that was `0q` becomes `-1q`. Iterate through candidate keys rather than hardcoding:

```python
def get_revenue_estimate(ticker_obj):
    try:
        re = ticker_obj.revenue_estimate
        for key in ("-1q", "0q", "+1q"):
            if key in re.index:
                val = re.loc[key, "avg"]
                if pd.notna(val):
                    return float(val)
    except Exception:
        pass
    return None
```

## Actual Revenue (`Ticker.quarterly_income_stmt`)

Returns a DataFrame with quarters as columns and line items as rows. To get actual revenue for a specific reporting quarter, match the closest column date to the earnings date.

**Gotcha, banks and insurers:** The `Total Revenue` row is often NaN for the most recent quarter even when other line items are populated. Fall through a chain of candidate rows:

```python
REVENUE_ROWS = ["Total Revenue", "Operating Revenue", "Net Interest Income"]

def get_actual_revenue(qis, earnings_date):
    if qis is None or qis.empty:
        return None
    # Try the 3 nearest period-end dates on or before earnings_date
    candidate_cols = sorted(
        [c for c in qis.columns if c <= earnings_date],
        reverse=True,
    )[:3]
    for col in candidate_cols:
        for row in REVENUE_ROWS:
            if row in qis.index:
                val = qis.loc[row, col]
                if pd.notna(val):
                    return float(val)
    return None
```

## Analyst Recommendations (`Ticker.recommendations`)

Returns a small DataFrame with rows keyed by relative month (`0m`, `-1m`, `-2m`, `-3m`) and columns `strongBuy`, `buy`, `hold`, `sell`, `strongSell`.

**This is a monthly rollup, not a per-action upgrade/downgrade feed.** Older yfinance exposed individual analyst actions; current versions do not. To detect "direction of change," compare current month to one month prior:

```python
def rec_trend(ticker_obj):
    recs = ticker_obj.recommendations
    if recs is None or recs.empty or len(recs) < 2:
        return None
    now = recs.iloc[0]  # 0m
    prev = recs.iloc[1]  # -1m
    return {
        "buy_change": int(now["strongBuy"] + now["buy"]) - int(prev["strongBuy"] + prev["buy"]),
        "sell_change": int(now["strongSell"] + now["sell"]) - int(prev["strongSell"] + prev["sell"]),
    }
```

**`recommendationMean` can be null:** Berkshire Hathaway (BRK-B) publishes no sell-side rollup, for example. Treat null as a known-source gap, not a failure.

## Price History (`Ticker.history`)

Use `start=` and `end=` with pandas Timestamps. Returns daily OHLCV plus `Dividends`, `Stock Splits`.

```python
hist = ticker_obj.history(start=earnings_date - timedelta(days=2),
                          end=earnings_date + timedelta(days=10))
# hist.index is tz-aware (usually America/New_York for US equities)
```

For "next trading day" and "5 trading days later" reactions, use positional indexing into the sorted DataFrame (not calendar-day math) so weekends and holidays are handled.

## Sector Taxonomy

Yahoo's `info["sector"]` differs from GICS:

| Yahoo | GICS |
|-------|------|
| Technology | Information Technology |
| Healthcare | Health Care |
| Financial Services | Financials |
| Consumer Cyclical | Consumer Discretionary |
| Consumer Defensive | Consumer Staples |
| Basic Materials | Materials |

If you need GICS-clean output, maintain an explicit mapping. If the source gives you GICS sectors separately (S&P index pages, Wikipedia), prefer that and keep Yahoo's `industry` as a secondary field.

## Ticker Format

yfinance uses **dashes** for share classes (`BRK-B`). This matches SEC's `company_tickers.json` but differs from Wikipedia's dotted form (`BRK.B`). Normalize to dashes when joining with yfinance data.
