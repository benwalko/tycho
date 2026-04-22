# Analysis -- Deliverables and Patterns

**Read `base.md` before starting. Technical patterns are in `.claude/agents/analyst.md`.**

## When to Run Analysis

Analysis runs AFTER the scraper agent completes and QC passes. It transforms raw data (`deliverable.xlsx`) into insights (charts, PDF, enhanced Excel).

The orchestrator spawns the analyst when:
- The requirements asked for analysis, charts, or a narrative
- The dataset is large enough to warrant summary views
- The data shape suggests visualization (geographic, temporal, numerical distribution)

Trivial or exploratory scrapes don't need an analyst run; a clean `deliverable.xlsx` plus `summary.txt` is enough.

## Deliverables

Single consistent output. See `.claude/agents/analyst.md` for the canonical list. Summary:

- **Enhanced Excel** (always): Summary / Analysis / Charts sheets added to the existing `deliverable.xlsx`. Data sheet is never modified.
- **Chart PNGs** (always): `output/charts/` directory, 150 DPI.
- **Updated summary.txt** (always): appended analysis section.
- **PDF report** (when data supports a narrative): executive summary, key metrics, interpreted charts, data quality notes.
- **Geographic map** (when location data is present): `output/map.html`.

Scope is deliberately bounded. Clustering, segmentation, automated EDA, and scoring are skipped unless the prompt explicitly asks.

## Patterns by Data Shape

Match patterns to what the data actually contains. See `.claude/agents/analyst.md` for the full shape-based taxonomy (categorical, geographic, temporal, numerical, completeness). Quick reference:

- **Categorical** → top-N breakdowns, distribution charts, cross-tabs
- **Geographic** → state/metro distribution, density heatmap, coverage-gap analysis
- **Temporal** → trend lines, YoY comparisons
- **Numerical** → histograms, summary stats, z-score anomaly detection, grouped comparisons
- **Completeness** → per-field coverage, composite score, low-completeness flags

## Chart Conventions

- Use seaborn style for consistency
- Title, axis labels, annotations
- PNG at 150 DPI, `bbox_inches='tight'`
- Pie charts: group slices under 3% into "Other"
- Don't rely on red/green alone for meaning (accessibility)

## Writing Rules for Analysis Outputs

The project's voice lives in `skills/voice.md` and applies to every analysis output. Analysis-specific additions:
- Executive summary readable by non-technical readers.
- If a chart needs explanation, one sentence below it is enough.
