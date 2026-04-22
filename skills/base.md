# Base -- Universal Operating Standards

**Read this before starting ANY job.** Every agent (researcher, scraper, analyst) must follow the rules in this file.

**Technical methodology** (recon phases, schema validation, circuit breaker, IP safety, anti-bot, escalation) lives in the agent templates at `.claude/agents/`. Reusable code lives in `skills/toolbox/`. This file covers behavioral and operational rules only.

## Execution Rules (CRITICAL)

Apply to any agent that runs scripts (researcher, scraper, analyst).

Run Python scripts DIRECTLY with Bash: `Bash(command='python3 script.py', timeout=600000)`. The script runs synchronously and returns all output when done. Max timeout is 600000ms (10 min). If a script needs more than 10 minutes, break it into stages that each run under 10 minutes and checkpoint between them.

FORBIDDEN patterns (these waste tokens and time):
- NEVER use `sleep` commands to wait for anything
- NEVER spawn sub-agents to run scripts
- NEVER background a process and poll for completion
- NEVER do `sleep 300 && check_progress` loops
- NEVER use `run_in_background: true` on Bash calls for your scripts

Retry policy:
- If something fails, read the error, fix the script, retry. Max 2 retries on the same error.
- After 2 failures on the same approach, pivot to a different method (see "Don't Loop" below).
- If you hit a hard blocker (Cloudflare Turnstile, CAPTCHA, login wall), stop and report. Don't waste turns.

## Writing Rules (CRITICAL)

The project's voice (tone, banned vocabulary, patterns to avoid, things to do) lives in `skills/voice.md`. Read it before producing any prose for a human reader. The QC checklist in `skills/qc.md` audits output against the same file.

## Autonomy Model

You run autonomously. The operator reviews your output, not your process. Don't ask for permission mid-execution unless you hit a genuine blocker.

Operator time is limited. Every unnecessary question or mid-job check-in costs time they don't have. Make decisions, document them in methodology.md, and keep moving.

## "Only Stop If" Rules

Stop and flag the operator ONLY if:
- Login wall requiring credentials you don't have
- Unbreakable anti-bot after exhausting the full escalation path
- Scope is 10x larger than described (e.g. "500 listings" turns out to be 50,000)
- Target site's ToS explicitly prohibits scraping AND no API alternative exists

Everything else: keep going, document issues in methodology.md.

## Don't Loop -- Pivot

If an approach isn't working after 2-3 genuine attempts, **stop, step back, and try a different angle.** Do not keep retrying the same thing with minor variations.

Signs you're stuck in a loop:
- Same error 3+ times in a row
- Tweaking headers/selectors/timeouts without meaningful progress
- Scraper returns empty data repeatedly from different pages
- You're writing "let me try one more thing" for the third time

When you catch yourself looping:
1. **Stop.** Write down what you've tried and why it failed.
2. **Diagnose.** Is this a headers issue, a rendering issue, a site structure issue, or an anti-bot issue? Each needs a fundamentally different approach, not a retry.
3. **Switch tracks.** If HTTP requests aren't working, try API interception. If browser rendering isn't working, check if there's a mobile site or alternative endpoint. If parsing is failing, the page structure may differ from what you assumed. Re-examine it.
4. **If 3 genuinely different approaches all fail**, flag the operator with what you tried. That's a real blocker, not a loop.

The goal is forward progress, not persistence on a dead path. Two focused pivots beat ten retries.

## Resumability

Jobs can be interrupted at any time (accidental escape, crash, timeout). Build for resumability from the start:

- **Write results to disk incrementally.** Don't hold everything in memory until the end. Use the CheckpointScraper pattern from toolbox or save partial results to CSV/JSON periodically.
- **Track progress in execution_log.md.** Each snapshot should include enough context to resume: which URLs/pages are done, current record count, where to pick up.
- **Scraper code must detect prior progress.** Before starting, check for existing checkpoint files or partial output. If found, resume from where it left off. Don't re-scrape completed pages.
- **If resumed after interruption:** Read execution_log.md, find the last snapshot, assess what's already done, and continue from there. Don't start over.

## Quality Bar

- **Target:** 90%+ field coverage (if a field exists on the page, capture it for 90%+ of records)
- **Alert threshold:** anything below 80% requires an explanation in `summary.txt` (why the gap, not just that it exists)
- Zero duplicate rows
- Normalized data: consistent phone format, trimmed whitespace, standardized addresses
- No fabricated data. If a field is empty on the source, it's empty in the output

## Deliverable Standards

- **Primary output:** XLSX with formatted headers (bold, auto-width columns, header row freeze)
- **summary.txt:** Total records, field coverage percentages, gaps or anomalies, execution time
- **methodology.md:** Tools used, approach taken, challenges encountered, rate limiting applied
- All files go in the job's `output/` subdirectory

## What We CANNOT Do

The full list of hard blockers, "works with care" cases, and volume guidance lives in `docs/limitations.md`. Read it before accepting a job that touches a known-hard target. The short version:

- Cloudflare Turnstile, Akamai Bot Manager, PerimeterX, DataDome: hard no.
- Login walls without credentials, CAPTCHAs: hard no.
- Sites with explicit anti-scraping ToS (LinkedIn, Yelp, Facebook, Instagram, etc.): hard no.
- Sustained volume against well-protected commercial sites: technically possible, but proxy rotation isn't built in.

When you spot one of these during triage, stop and report. Don't start a job you can't finish.
