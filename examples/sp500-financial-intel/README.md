# Example: S&P 500 Financial Intelligence Dataset

A worked example showing how the framework handles a multi-source enrichment job.

**This is illustrative documentation, not reproducible code.** The scraper scripts were written by the agent per-run and are not checked in. This directory contains the original requirements, the agent's post-run execution log, and the final methodology/summary docs so you can see the full arc. The actual 503-row dataset, charts, and PDF are gitignored to keep the repo small. If you want to reproduce it, paste `requirements.md` into a fresh job and spawn a scraper agent.

## The Request

Build a single dataset covering all 503 S&P 500 companies by combining four public data sources into one Excel file with derived signals. The only constraint: zero API signups or paid subscriptions.

**Sources required:**
1. **Yahoo Finance** (via `yfinance` library) - price, fundamentals, analyst recs
2. **SEC EDGAR** (REST API) - insider transactions (Form 4) for the last 90 days
3. **Treasury.gov** - current yield curve
4. **BLS.gov** - sector-level employment trends

**Derived columns required:**
- Insider Sentiment Score (-1 to +1, weighted by dollar value)
- Earnings Yield vs Treasury Spread (earnings yield minus 10Y yield)
- Sector Employment Momentum (YoY employment change for company's sector)
- Composite Signal (Bullish / Mixed / Bearish)

## How the Framework Handled It

### 1. Orchestrator scaffolded the job

Created the job directory, wrote `requirements.md` with the full spec, set up `output/`.

### 2. Orchestrator spawned a scraper agent in the background

Spawned via `subagent_type: scraper` with `run_in_background: true`. Job-specific context in the prompt:
- Job directory path
- Reference to `requirements.md`
- Technical notes (yfinance install, SEC User-Agent requirements, BLS API endpoints)
- Contents of `skills/sites/sec-gov-edgar.md`

The agent ran autonomously for ~70 minutes while the orchestrator continued doing other work.

### 3. Agent executed in four stages

**Stage 1 (15s):** Foundation data.
- Pulled S&P 500 list from Wikipedia (503 companies). First attempt blocked (403) due to default urllib User-Agent. Fixed by switching to a browser UA.
- Tried Treasury.gov CSV and OData endpoints. Both returned empty data or HTML redirects. Pivoted to `yfinance` for market yield tickers (3mo, 5yr, 10yr, 30yr).
- Pulled BLS employment data for 10 of 11 GICS sectors.

**Stage 2 (144s):** Yahoo Finance.
- 503/503 tickers fetched, zero errors.
- 3.5 tickers/sec, no throttling needed.
- Captured price, P/E, market cap, dividends, analyst recs, margins, beta.

**Stage 3 (~570s total across retries):** SEC EDGAR.
- First attempt: submissions API for filing counts worked. XML parsing for transaction detail hit 429s on every attempt.
- Second attempt: stricter rate limiting (7 req/sec) and retry backoff. Still 429'd.
- Third attempt: pivoted to EFTS search API (efts.sec.gov), which turned out to have a SEPARATE rate pool from www.sec.gov. Got transaction detail for 74 companies.
- Fourth attempt: merged EFTS counts for remaining companies using the separate rate pool.

Final: 503 companies, 465 with filing counts, 74 with parsed transaction detail.

**Stage 4 (instant):** Assembly.
- Merged all sources, computed derived columns.
- Found that Yahoo Finance uses a different sector taxonomy than GICS ("Technology" vs "Information Technology", "Healthcare" vs "Health Care"). Added explicit mapping to get 100% BLS coverage.
- Formatted Excel with color-coded section headers, number formatting, frozen row, auto-filter.

### 4. Orchestrator ran QC

Ran through `skills/qc.md`:
- All deliverables present: yes
- Field coverage: 21 fields at 100%, 28 of 35 fields above 90%
- Insider Sentiment at 14.7% (explained: SEC rate limit constraint on XML parsing)
- Writing standards: caught the word "comprehensive" in methodology.md, fixed it (one-word banned-list edit)
- No em-dashes
- Cleaned up 7 intermediate staging scripts and checkpoint files

### 5. Orchestrator spawned an analyst agent

Spawned via `subagent_type: analyst` with `run_in_background: true`. Prompt included job directory and path to `deliverable.xlsx`. Agent ran in the background, produced:
- Enhanced Excel (added Summary, Analysis, Charts sheets)
- 10-page PDF report with executive summary
- 7 PNG charts at 150 DPI (signal by sector, P/E by sector, market cap distribution, earnings yield spread histogram, dividend vs P/E scatter, employment momentum, top 15 cheapest stocks)

### 6. Orchestrator reported results

The operator got a completion summary with the key findings and paths to deliverables.

## What the Framework Made Easy

1. **Autonomy.** Orchestrator wasn't blocked while the agent ran. Other work continued in parallel.
2. **Pivots.** The scraper agent pivoted approach three times on SEC EDGAR (direct filing fetch → stricter rate limiting → EFTS separate rate pool) without the orchestrator needing to intervene.
3. **Checkpointing.** If the agent had been killed mid-stage-3, it would have resumed from the last completed CIK.
4. **QC.** The orchestrator caught a writing-rules violation (one banned word) that the agent didn't, before the deliverable went to the operator.
5. **Knowledge capture.** The execution log documented the multi-rate-pool behavior of sec.gov vs efts.sec.gov, which now lives in `skills/sites/sec-gov-edgar.md` for future jobs.

## What Didn't Go Perfectly

1. **Treasury.gov endpoints were defunct.** Both CSV and OData feeds returned empty data. The agent pivoted to yfinance for market yields. The fallback (use `^TNX`, `^FVX`, etc.) is now documented in `skills/toolbox/yfinance.md` so future jobs skip the dead end.
2. **SEC rate limit recovery took multiple attempts.** The multi-minute cooldown behavior (not just per-second throttling) wasn't obvious until observed at scale. Now documented in `skills/sites/sec-gov-edgar.md`.
3. **Insider sentiment coverage was 14.7% instead of the target 100%.** The cost of parsing Form 4 XML for every company on www.sec.gov was too high given the rate pool issues. Acceptable tradeoff given the constraint.

## Files

- `requirements.md`: the original spec
- `execution_log.md`: the scraper agent's internal log
- `output/methodology.md`: the plain-language methodology doc
- `output/summary.txt`: coverage stats and composite signal distribution

The 503-row Excel, charts, and PDF aren't checked in (gitignored). To reproduce, paste `requirements.md` into a fresh job and spawn a scraper.

## Key Takeaways

- Multi-source enrichment is a natural fit for a single background agent. Breaking it into separate agents per source would have added complexity without benefit.
- Rate-limit knowledge compounds. The "efts.sec.gov has a separate rate pool" insight wouldn't have been discovered without the first two failed attempts. That's now a permanent part of the site profile.
- The orchestrator's QC step is cheap insurance. One banned-word catch is low cost, but represents the kind of polish that distinguishes a ready deliverable from a rough one.
- Pivots matter more than persistence. Four stages, three pivots, one successful result.
