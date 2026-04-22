# Troubleshooting

When a job goes sideways, here's where to look first.

## Where the evidence lives

Every job leaves a paper trail in `jobs/{slug}/`:

| File | What it tells you |
|------|-------------------|
| `job.json` | Status (`active`, `completed`, `failed`), timestamps, record counts, failure reason |
| `execution_log.md` | The agent's narration: approaches tried, pivots taken, rate limits observed, what worked, what didn't |
| `scripts/` | The actual Python the agent wrote and ran. The fastest way to understand what happened. |
| `output/` | Final deliverables. If something looks wrong here, the script in `scripts/` is the next place to look. |
| `output/summary.txt` | Field-coverage percentages and quality notes. Low coverage is usually explained at the bottom. |

## Common situations

### "The agent said it's running but I don't see anything happening"

Background agents emit one notification at completion, not progress updates. While they run, the only signs of life are inline status if you ask the orchestrator (`What's the agent doing?`).

For longer jobs you can also `ls jobs/{slug}/` and check `scripts/checkpoint.json` size to confirm progress is hitting disk.

### Agent reported BLOCKED

The orchestrator will surface the structured failure report (`skills/failure-template.md` format). Read it. Decide one of three things:

- **Retry with a different angle.** The agent may have missed an alternate endpoint, mobile site, or adjacent API. Spawn a fresh agent with the new direction in the prompt.
- **Escalate.** You may have credentials, domain knowledge, or context the agent didn't.
- **Drop the job.** Some sites aren't worth the effort. Document the dead end in the site profile so the next person doesn't repeat it.

### Output looks wrong

1. Open `output/summary.txt` and check field coverage. Anything below 80% should have an explanation at the bottom; if it doesn't, that's where to dig.
2. Open `execution_log.md` and skim the "Validation Results" or similar section.
3. Spot-check 5-10 rows in `deliverable.xlsx` against the source URL. If the source has data the deliverable is missing, the scraper logic is the bug.
4. The script that produced it is in `scripts/`. The naming convention is usually numbered (`01_recon.py`, `02_scrape.py`, `03_build_output.py`).

### PDF narrative disagrees with the numbers

Hardcoded narrative in chart captions can drift from the computed stats it sits next to. We have a Narrative Rule in `.claude/agents/analyst.md` that requires prose to be computed or cross-checked, but it's worth a sanity-check on first read.

If the narrative is wrong, the fix is in `scripts_analysis/build_pdf.py`. Edit the offending string, re-run `python3 scripts_analysis/build_pdf.py`. The `output/analysis_report.pdf` will regenerate.

### Job died mid-run (Claude Code closed, machine restarted)

Background agents live inside the Claude Code session. When the session dies, so do the agents.

Recovery:

1. Restart `claude` from the repo root.
2. Tell the orchestrator: `Resume the job at jobs/{slug}/`. It'll inspect `job.json`, `execution_log.md`, and `scripts/checkpoint.json` to figure out where to pick up.
3. Spawn a fresh scraper agent against the same job directory. The agent's first action should be to detect the existing checkpoint and continue from there. If it doesn't, point it at the checkpoint explicitly in the prompt.

### Rate-limit pain mid-job

Most rate limits are documented in the relevant `skills/sites/*.md` profile. If the agent is getting 429s:

1. Check whether the profile's `rate_limit_rps` matches what the agent is doing. If the agent is faster than the profile says is safe, that's the bug.
2. Some sites have multi-pool rate limits (different endpoints, different counters). The SEC profile (`skills/sites/sec-gov-edgar.md`) documents this for `efts.sec.gov` vs `www.sec.gov`. Check whether the agent is exceeding the pool it's actually hitting.
3. If the rate limit is new (the profile doesn't reflect it), let the run finish at a slower pace, then update the profile during the retrospective.

### Scripts directory is empty or missing

Per convention, the scraper writes to `jobs/{slug}/scripts/` and the analyst to `jobs/{slug}/scripts_analysis/`. If either is empty, the agent likely failed before producing output, or skipped the directory entirely.

Check `job.json`. If `status` is `failed`, read `failure_reason` and `execution_log.md`. If `status` is `completed` but the directory is empty, that's a real bug; spawn a follow-up agent or re-run the job.

## When to ask the orchestrator vs. dig in yourself

- Ask the orchestrator first for narrative: "What did the scraper try?" "Why did it pivot?" "Why is field X at 60%?"
- Dig in yourself when you need the literal source: open `scripts/`, `execution_log.md`, or `output/deliverable.xlsx` directly. Reading the actual code or output is faster than asking the orchestrator to summarize.
