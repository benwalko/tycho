# Example: S&P 100 Earnings Season Monitor

A worked example showing the framework on a focused, time-bounded multi-source job. Smaller scope than the SP500 example, more representative of a typical day's work.

**Illustrative documentation, not reproducible code.** The scraper and analyst scripts were generated per-run and aren't checked in. The `deliverable.xlsx`, chart PNGs, and PDF report are gitignored. To reproduce, paste `requirements.md` into a fresh Claude session and spawn a scraper.

## The Request

Build a single Excel covering the most recent earnings release for every S&P 100 company over the past 30 days. Combine fundamentals, filings metadata, analyst activity, and macro rate context. Public sources only.

**Sources:**
1. **Yahoo Finance** via `yfinance`: company metadata, earnings dates and surprises, post-earnings price reactions, analyst recommendation rollups
2. **SEC EDGAR**: 10-Q filing metadata for each reporter
3. **`^TNX`** via yfinance: 10-year Treasury yield for valuation context

**Derived columns:**
- 1-day and 1-week price reactions (computed from daily closes)
- EPS Beat/Miss flag, Revenue Beat/Miss flag, Double Beat flag
- Earnings yield vs 10Y spread (basis points)
- Reaction-vs-Surprise alignment (Aligned Positive / Aligned Negative / Contra)

## How the Framework Handled It

### 1. Orchestrator scaffolded the job

Created the job directory, wrote `requirements.md` with the full spec, set up `output/`. No researcher run needed: SEC EDGAR has a fresh profile in `skills/sites/sec-gov-edgar.md`, and Yahoo Finance is consumed via the `yfinance` library, not by scraping the site.

### 2. Orchestrator spawned the scraper agent

Spawned via `subagent_type: scraper` with `run_in_background: true`. Job context in the prompt:

- Job directory path
- Reference to `requirements.md`
- Contents of `skills/sites/sec-gov-edgar.md`
- Pointer to `skills/toolbox/yfinance.md` for library quirks
- Treasury.gov-is-defunct note (use `^TNX` instead)

The agent ran for ~10 minutes.

### 3. Agent execution

Single-pass scrape across all 101 constituents (S&P 100 includes both GOOG and GOOGL share classes):

- **Wikipedia**: pulled the constituent table. Default urllib UA blocked (403); switched to a browser UA.
- **SEC `company_tickers.json`**: one request, mapped tickers to CIKs. Discovered SEC stores share-class tickers with dashes (`BRK-B`), not dots (`BRK.B`) like Wikipedia. Added a normalization step.
- **yfinance per ticker**: pulled `info`, `earnings_dates`, `recommendations`, `revenue_estimate`, `quarterly_income_stmt`, and `history` around the earnings date. 0.25s pacing per ticker. No throttling at this scale.
- **SEC `data.sec.gov/submissions/...`**: one request per CIK to find 10-Q filings within the window. 0.3s pacing, zero 429s across 101 sequential requests.
- **`^TNX`**: one call for the current 10Y Treasury yield.

Two pivots during the run:

- **Revenue lookup for banks**. Initial code only checked `Total Revenue` in the quarterly income statement. JPM and other banks had NaN there. Pivoted to a fallback chain: `Total Revenue` → `Operating Revenue` → `Net Interest Income`. Final coverage among reporters: 100%.
- **CIK lookup format**. First attempt used Wikipedia's dotted ticker form, which SEC doesn't recognize. Fixed by trying yfinance form (dashes) first.

### 4. Orchestrator ran QC

- All deliverables present
- Field coverage: 100% on ticker / company / sector / forward P/E / CIK; 99% on analyst rec mean (BRK.B has none)
- Earnings-related fields at 96-100% coverage among reporters; structural gaps (price +1D, +5D, 10-Q filing date) explained in `summary.txt`
- Caught two em-dashes in `execution_log.md`, fixed inline

### 5. Orchestrator spawned the analyst agent

Spawned via `subagent_type: analyst`. Produced:

- Enhanced `deliverable.xlsx` with Summary, Analysis, Charts sheets
- 9 PNG charts at 150 DPI: sector distribution (full and reporters), EPS outcomes pie, 1-day reaction histogram, forward P/E histogram, earnings yield spread histogram, EPS surprise vs reaction scatter, alignment counts, beat rate by sector
- `analysis_report.pdf` with cover, executive summary, key metrics, interpreted charts, contra table, top/bottom 5 tables, methodology

Caught one quality issue during analyst QC: a hardcoded chart caption claimed "slight positive skew" when the actual distribution had a `-15.5%` outlier. Fixed in the source script and regenerated the PDF. This led to a new "Narrative Rule" in `.claude/agents/analyst.md` requiring distribution prose to be computed or cross-checked.

### 6. Post-job retrospective

Updates to the knowledge base:

- `skills/sites/sec-gov-edgar.md`: bumped `last_probed`, added the BRK-B dash-vs-dot gotcha
- `skills/toolbox/yfinance.md`: created (new file) with the library patterns surfaced during this run: Treasury tickers, earnings_dates handling, revenue_estimate row-key shifts, recommendations monthly pivot, bank revenue fallback chain, sector taxonomy mapping, ticker normalization
- `skills/toolbox/README.md`: added the new module to the index
- `jobs/README.md`: documented the `scripts_analysis/` convention the analyst introduced
- `.claude/agents/analyst.md`: added the Narrative Rule

## Headline Findings

- 29 of 101 reported in window; 23 had posted actuals by scrape time
- 62.1% EPS beat rate across all reporters (78.3% among classified)
- 1-day reaction distribution centered just below zero (median -0.6%, mean -1.3%); one reporter posted a ~-15% loss, pulling the mean down
- Only ~30 of 101 constituents offer an earnings yield above the 4.288% 10Y Treasury; mega-cap valuations are tight
- Financials drove the early-season reporter mix (calendar effect)

## What the Framework Made Easy

1. **Profile reuse.** The SEC EDGAR profile from prior jobs let the scraper skip rediscovering rate limits and User-Agent requirements.
2. **Toolbox compounding.** The yfinance patterns surfaced here now live in a dedicated toolbox module, so the next earnings-style job won't relitigate them.
3. **Single analyst pass.** One spawn produced the enhanced Excel, charts, and PDF. No tier selection, no per-job decisions about scope.
4. **QC catching narrative drift.** The analyst's hardcoded chart caption disagreed with its own computed numbers. The QC pass caught it before the PDF reached the operator.

## What Didn't Go Perfectly

1. **Hardcoded prose vs computed stats.** The analyst's "slight positive skew" caption was wrong. Surfaced a real systemic risk in any auto-generated narrative; the new Narrative Rule addresses it.
2. **`scripts_analysis/` was an undocumented convention.** The analyst chose a sensible directory name; the docs didn't anticipate it. Now formalized in `jobs/README.md` and `.claude/agents/analyst.md`.

## Files

- `requirements.md`: the original spec
- `execution_log.md`: the scraper's internal log
- `execution_log_analysis.md`: the analyst's internal log
- `output/methodology.md`: plain-language writeup
- `output/summary.txt`: coverage stats and analysis section

The actual `deliverable.xlsx`, chart PNGs, and PDF report are gitignored to keep the repo small.

## Key Takeaways

- Multi-source enrichment with reused site profiles is materially faster than starting from scratch. The SP100 run finished in roughly half the time per record of the comparable SP500 run, mostly because the SEC EDGAR knowledge was already captured.
- Toolbox modules emerge from real jobs. `yfinance.md` exists because two jobs hit the same library quirks; one job wouldn't have justified it.
- Auto-generated narrative needs to be derived from data, not eyeballed. Charts and prose drifting apart is a real failure mode worth a hard rule.
