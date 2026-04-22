# Analysis Execution Log - S&P 100 Earnings Season Monitor

## Context

Scraper produced a validated `output/deliverable.xlsx` with 101 rows (one per S&P 100 constituent) and 40 columns spanning fundamentals, earnings outcomes, price reactions, valuation, analyst sentiment, and SEC 10-Q metadata. 29 companies reported earnings in the 30-day window ending 2026-04-22. Treasury 10Y (^TNX) snapshot: 4.288%.

Sheet containing raw data is named `SP100 Earnings` (not `Data`). Confirmed via openpyxl before reading so I did not clobber the scraper's tab naming.

## Approach

One script per concern, kept in `scripts_analysis/`:
- `build_charts.py` - matplotlib/seaborn at 150 DPI, seaborn-whitegrid style, husl palette keyed by sector for consistency across charts.
- `build_excel.py` - loads the existing workbook, deletes any existing Summary/Analysis/Charts tabs, re-appends them. Idempotent: safe to re-run.
- `build_pdf.py` - reportlab PDF with cover, 3-paragraph executive summary, key metrics table, interpreted charts, contra table, top/bottom 5 tables, data quality notes, and a methodology paragraph.
- `append_summary.py` - appends an Analysis section to `summary.txt`. Idempotent: strips any prior appended block via a marker line before rewriting.

## Analytical choices

- **Reporter universe versus full universe**: Every aggregate makes this split explicit. Sector distribution is shown for both (101 and 29). Beat/miss, reactions, alignment, and top/bottom 5 operate on reporters only. Forward P/E and earnings yield spread operate on the full 101.
- **Classified vs reporters**: 29 reported, but only 23 have a definitive EPS Beat/Miss classification (6 same-day reports had no actual at scrape time). Beat rate is stated as "of classified" in the written deliverables so the arithmetic is transparent (n_beat=18, classified=23 -> 78.3% of classified, or 62.1% of all reporters). The numerical summary in the existing `summary.txt` uses the 62.1% denominator; I mirrored that wording and also surfaced the 78.3% number in the exec summary for completeness.
- **Structural gaps are annotated, not flagged as quality issues**: Price +1D (22/29), Price +5D (9/29), and 10-Q filing date (8/29) appear in the coverage tables with a note explaining why.
- **Contra bucket**: Surfaced as a standalone table in both the Analysis sheet and the PDF; ordered by 1-day reaction ascending so the biggest negative dislocations read first.
- **P/E chart clipped at 60x**: Long tail on Forward P/E compresses the histogram; clipped to 60 with that stated in the title and left a median line.
- **No geographic map**: Confirmed no lat/lng fields exist.
- **No custom scoring, clustering, or EDA profile**: Not in scope.

## Deliverables produced

- `output/deliverable.xlsx` - appended three sheets:
  - `Summary`: key counts, numerical summary (7 metrics), field coverage for full universe and reporters, notes.
  - `Analysis`: sector distribution table, beat/miss counts, beat rate by sector, alignment counts, contra table, top/bottom 5 tables for 4 metrics, earnings yield spread summary.
  - `Charts`: 4 native openpyxl charts (bar, pie, bar, column) plus embedded PNG previews of histograms and the scatter.
- `output/charts/` - 9 PNGs at 150 DPI.
- `output/analysis_report.pdf` - cover + exec summary + key metrics + 9 interpreted charts + contra table + top/bottom 5 tables + data quality + methodology.
- `output/summary.txt` - appended Analysis section with beat/miss math, reaction stats, alignment counts, valuation stats, reporters by sector, and a deliverables manifest.

## Key findings

- 62.1% EPS beat rate across all 29 reporters (78.3% among the 23 with classifications posted); 10.3% miss rate; 8 double beats.
- Median 1-day reaction is -0.6%, mean -1.3% (n=22 with post-earnings price data). One reporter posted a ~-15% single-day loss, pulling the mean down; most reactions cluster within +/-3%.
- Of 22 classifiable 1-day reactions, 12 showed contra moves (price moved opposite the EPS surprise sign), underscoring how guidance and segment detail override headline beats/misses.
- Forward P/E skews high across the S&P 100. Only ~30 of 101 constituents offer an earnings yield above the 10Y Treasury; the rest trade at or below parity with the risk-free rate.
- Financials account for a disproportionate share of early-season reporters, which is a calendar effect, not a screening result.

## Files produced by the analysis pass

- `scripts_analysis/build_charts.py`
- `scripts_analysis/build_excel.py`
- `scripts_analysis/build_pdf.py`
- `scripts_analysis/append_summary.py`
- `output/charts/01_sector_distribution.png` through `09_beat_rate_by_sector.png`
- `output/analysis_report.pdf`
- `output/deliverable.xlsx` (three appended sheets; Data sheet untouched)
- `output/summary.txt` (Analysis section appended)
