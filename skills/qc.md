# QC -- Quality Control Checklist

**Trigger:** After every agent completes a job, before reporting results to the operator.

The orchestrator runs this checklist against the agent's output. If any check fails, fix it (spawn a follow-up agent or fix manually) before reporting results.

## Deliverable Completeness

- [ ] `output/deliverable.xlsx` exists and is non-empty
- [ ] `output/summary.txt` exists with row counts and field coverage
- [ ] `output/methodology.md` exists, describes approach
- [ ] `job.json` validates against the schema in `jobs/README.md`: `status` is `completed`, `completed_at` is set, `records` is populated
- [ ] `execution_log.md` exists in job root

## Data Quality

Read `summary.txt` and check:
- [ ] Row count matches what requirements.md specified (or is reasonably close with explanation)
- [ ] No field below 80% coverage without an explanation in the summary
- [ ] Quality notes section addresses any sparse fields (why, not just what)

## Writing Standards

Audit `methodology.md`, `summary.txt`, and any other prose deliverable against `skills/voice.md`:
- [ ] No em-dashes
- [ ] No banned vocabulary (see voice.md for the full list)
- [ ] No filler transitions (Furthermore, Additionally, It's worth noting, etc.)
- [ ] Active voice by default. Numbers over adjectives. Short to medium sentences.
- [ ] No SaaS-marketing phrasing. Use the quick test in voice.md when in doubt.

## Execution Log

Read `execution_log.md`:
- [ ] Documents what approach was used and why
- [ ] Includes recon results (what Phase 0 found)
- [ ] If pivots happened, explains what triggered them
- [ ] Includes specific numbers (request counts, rate limits observed, timing)
- [ ] Has recommendations for future scrapes of this site

## Cleanup

- [ ] No `checkpoint.json` left in job directory
- [ ] No `__pycache__/` directories anywhere in the job
- [ ] `scripts/` contains only the scripts that actually ran. Throwaway/debug scripts either deleted or prefixed with `_` or `test_` so their status is obvious.
- [ ] No raw data dumps left outside `output/` (raw CSVs, JSON dumps used during processing)

## Analysis Deliverables (if analysis was included)

- [ ] Enhanced Excel: Summary/Analysis/Charts sheets exist, Data sheet untouched
- [ ] Chart PNGs in `output/charts/` are readable (not too small, labels not cut off, no default "Figure 1" titles)
- [ ] PDF report opens correctly and images render (if produced)
- [ ] Executive summary is written in plain English, no jargon (if PDF produced)
- [ ] Geographic map loads in a browser (if location data warranted one)
- [ ] Chart colors are accessible (not relying on red/green alone)

## What To Do When QC Fails

- **Missing deliverable:** Spawn a follow-up agent to generate it
- **Bad writing:** Fix it directly (small edits don't need an agent)
- **Missing execution_log.md:** Write one retroactively from the agent's scripts and output
- **Leftover files:** Clean up directly
- **Data quality issue:** Flag to operator. Don't ship bad data.
