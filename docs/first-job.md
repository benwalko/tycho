# Your First Job

A 5-minute walk-through. By the end you'll have a real Excel deliverable on disk and a working sense of how the framework operates.

## Prerequisites

You've worked through the [Setup section](../README.md#setup) in the main README. Specifically: Claude Code is installed and authenticated, Python 3.10+ is on your PATH, and you've cloned this repo.

## Step 1: Start the orchestrator

From the repo root:

```bash
claude
```

The orchestrator is the Claude Code main session. It reads `CLAUDE.md` automatically, which loads the framework into its context. You don't need to set anything else up.

## Step 2: Send the prompt

Paste this verbatim:

> Scrape the S&P 100 constituents list from Wikipedia. For each ticker, also pull market cap, sector, and forward P/E from Yahoo Finance via the yfinance library. Output as a single Excel.

This is a small job (~100 records, two well-known sources, no anti-bot) so it makes a good first run. Expect 2 to 4 minutes total.

## Step 3: What you'll see

The orchestrator works through this in roughly five steps:

1. **Feasibility check.** It reviews the request against `skills/sites/`. Wikipedia and yfinance are both well-known to the framework, so it skips a researcher run.
2. **Scaffold.** It creates `jobs/sp100-basic/` (slug may vary) with `job.json`, `requirements.md`, and an empty `output/`.
3. **Spawn scraper.** You'll see something like `Async agent launched successfully`. The scraper agent is now running in the background.
4. **You can keep talking.** The orchestrator stays responsive. Try asking `What's the agent doing?` and it'll give you a status read.
5. **Completion.** When the scraper finishes, the orchestrator gets a notification and reports back: row count, field coverage, paths to deliverables.

## Step 4: Inspect the output

```bash
open jobs/sp100-basic/output/deliverable.xlsx
```

You should see one row per S&P 100 constituent with ticker, company, sector, market cap, and forward P/E.

Other files in the job directory:

| File | What's in it |
|------|--------------|
| `output/summary.txt` | Coverage stats, row counts, field completeness |
| `output/methodology.md` | Plain-language writeup of how the data was sourced |
| `execution_log.md` | The agent's internal log: what it tried, what worked, anything surprising |
| `scripts/` | The Python scripts the agent actually wrote and ran |
| `job.json` | Status, timestamps, record counts |

The `scripts/` directory is worth a look. It's the actual code that ran. Reading it is the fastest way to understand what the agent did.

## Step 5 (optional): Add analysis

In the same Claude session:

> Run analysis on this dataset

The orchestrator spawns the analyst agent. After a few minutes you'll have:

- Three new sheets in `deliverable.xlsx` (Summary, Analysis, Charts)
- `output/charts/` with PNG charts
- `output/analysis_report.pdf` if the data shape supports a narrative

## What to try next

- **Probe a new site.** Ask the orchestrator: `Run a recon on data.cms.gov`. The researcher agent will probe the site and write a profile to `skills/sites/`. Useful before committing to a real job on an unfamiliar target.
- **Multi-source enrichment.** See `examples/sp500-financial-intel/` for a worked example combining Yahoo Finance, SEC EDGAR, Treasury yields, and BLS employment.
- **Read the limits.** `docs/limitations.md` covers what the framework can't do, so you don't burn a session on an unscrapable target.

## If something goes wrong

- **Agent reports BLOCKED.** It hit a hard limit (anti-bot, ToS, missing credentials). The orchestrator will surface the structured failure report. Decide: try a different angle, scope down, or drop the job.
- **Agent runs longer than expected.** Background agents need the Claude Code session to stay open. Long jobs can be 30+ minutes. If you have to close the session, the agent dies; restart by spawning a fresh one against the same job directory and it'll resume from the last checkpoint.
- **Output looks off.** The orchestrator runs QC against `skills/qc.md` after every job and fixes small issues directly. If you spot something it missed, tell it; it'll launch a follow-up agent to fix it.
