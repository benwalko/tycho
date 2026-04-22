# Tycho

An agentic scraping framework that runs end to end inside a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) session. You describe what you want in plain language. An orchestrator spawns specialized agents to handle recon, scraping, and analysis, then drops the deliverables in a job directory.

What makes Tycho different is what happens after the job ends. Every job writes back what it learned to a shared knowledge base:

- **Site profiles** record what worked and what didn't on each domain, including rate limits observed at scale.
- **The toolbox** accumulates reusable code patterns that survive across jobs.
- **Skill files** absorb new methodology when a job surfaces a gap in guidance.

The first scrape of a site is cold. The tenth is warm. See [Knowledge Base](#knowledge-base) for how the compounding loop is structured.

Named for Tycho Brahe, the Danish astronomer whose four decades of patient, pre-telescope observation built the catalog that Kepler used to derive the laws of planetary motion. The framework gathers data the same way: from multiple angles, into a shared record that outlives any single job.

Multi-source enrichment is the sweet spot: combining a few clean public sources into one deliverable that didn't exist before.

> Claude Code is required. The framework relies on Claude Code's named-subagent mechanism, background-agent execution, and `CLAUDE.md` auto-loading. It's not designed to run on other agent runtimes.

## Architecture

```
You (operator) → Orchestrator → Sub-agents → Deliverables
```

- **Orchestrator** runs the job, spawns agents, runs QC, updates the knowledge base
- **Researcher agents** probe unknown sites and write a reusable site profile
- **Scraper agents** write the script, run it, produce structured data
- **Analyst agents** turn raw data into charts, PDFs, and an enhanced Excel

The orchestrator never scrapes. Agents never make strategic calls. Each role stays in its lane, which keeps each prompt short and each job recoverable.

## Why This Exists

Most scraping work is one site at a time, scripts in someone's repo, knowledge in someone's head. Multi-source enrichment usually means stitching together separate downstream pipelines after the fact. This framework runs the whole job in one session: recon, scraping, analysis, writeup. Every learning lands in shared files so the next job benefits.

Over time, the knowledge base becomes the part of the repo that's hardest to replace. The code can be rewritten; an accumulated set of site profiles built from actual probing cannot.

## Project Structure

```
.claude/agents/          # Agent prompt templates (researcher, scraper, analyst)
skills/                  # Operational playbooks and methodology
  base.md               # Universal operating rules
  voice.md              # The project's writing voice (tone, banned vocab, patterns)
  extraction.md         # Scraping techniques (pagination, volume, dynamic content)
  analysis.md           # Analysis deliverables (Excel enhancement, PDF reports, charts)
  qc.md                 # Quality control checklist
  selenium.md           # Browser automation patterns
  failure-template.md   # Structured blocker-report format for agents
  sites/                # Per-domain technical profiles (what works, what doesn't)
  toolbox/              # Reusable code recipes (rate limiting, checkpoints, cleaning)
jobs/                   # Job directories (created per-project, gitignored)
examples/               # Worked examples with write-ups
docs/                   # Setup, first job walkthrough, limitations, architecture
  architecture.md       # Multi-agent design and the job lifecycle
  first-job.md          # Step-by-step walk-through of a 5-minute first scrape
  limitations.md        # What the framework can't do, and what to use instead
  troubleshooting.md    # Where to look when a job goes sideways
CLAUDE.md               # The orchestrator's operational guide (auto-loaded by Claude Code)
```

## Setup

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (required; the framework is built around its named-subagent and background-agent mechanisms).
- macOS or Linux (Windows works via WSL).
- Python 3.10 or newer (`python3 --version`). Agents install packages with `pip` as needed.
- An [Anthropic API key](https://console.anthropic.com/) with billing enabled.

### Install Claude Code

Follow the [official install instructions](https://docs.anthropic.com/en/docs/claude-code/quickstart). Once it's installed:

```bash
claude  # first run prompts for the API key and a workspace trust check
```

### Permissions

Agents in this framework need to install Python packages and run scripts. Each agent template in `.claude/agents/` declares its tool list (Bash, Read, Write, etc.) and uses `permissionMode: bypassPermissions` so agents don't pause on every shell call. Look over the templates if you want to know exactly what each agent has access to.

If your Claude Code config has stricter defaults (organization policy, locked-down profile), add this repo to your trusted-projects list in `~/.claude/settings.json`.

### Cost expectation

Each job consumes API tokens proportional to: recon turns, scraper turns (driven by record count, retries, and pivots), and analyst turns. A small single-source scrape is cheap. A multi-source enrichment with retries is materially more. Check the Anthropic console after your first few runs to calibrate. For reference, the SP100 example in `docs/first-job.md` finishes in ~135K total tokens across two agents.

## Quick Start

```bash
git clone git@github.com:benwalko/tycho.git
cd tycho
claude
```

Once `claude` is running, describe what you want at the prompt:

> Scrape all S&P 500 company data from Yahoo Finance. I want ticker, price, P/E, market cap, dividend yield, and sector for every company.

The orchestrator handles feasibility, scaffolding, scraper spawn, QC, and (if the data warrants) analysis. You'll end up with `jobs/{slug}/output/deliverable.xlsx` plus a summary, methodology, and chart pack.

For a step-by-step first run with an exact prompt and expected output, see [`docs/first-job.md`](docs/first-job.md). Before pitching a hard target, skim [`docs/limitations.md`](docs/limitations.md).

### Recon without scraping

When you're sizing up a new site before committing to a full job, ask for recon directly:

> Run a recon on zillow.com. I want to know whether we can pull active listings at scale.

The orchestrator spawns the researcher agent, which writes a profile to `skills/sites/zillow-com.md` and reports back a viability verdict (green / yellow / red) along with the recommended approach and observed rate limits.

### Long-running jobs

Background agents live inside the Claude Code session. While an agent runs:

- Keep Claude Code open. Closing the terminal or killing the session terminates the agent.
- You can keep talking to the orchestrator. It stays responsive and notifies you inline when each agent completes.
- For jobs over an hour, plan to leave the session running. Agents checkpoint to disk, so a killed run can be restarted by spawning a fresh agent against the same job directory; it picks up from the last checkpoint.

## How Agents Work

Agent templates live in `.claude/agents/`. Each file defines one agent's contract: what to read first, what to produce, how to handle failures. Claude Code loads each file as a named subagent type.

The orchestrator spawns an agent by passing its name as `subagent_type` (`researcher`, `scraper`, or `analyst`) along with `run_in_background: true`. Job-specific context goes in the prompt string. The agent works on its own and reports back when it's done.

### Researcher Agent

- Recon only; does not produce a dataset
- Hits the target with several UA profiles, checks for anti-bot, looks for embedded JSON and APIs, inspects pagination
- Writes a reusable profile to `skills/sites/{domain}.md` and updates the site index
- Returns a viability verdict (green / yellow / red) so the orchestrator can decide whether to take the job

### Scraper Agent

- Runs abbreviated recon when a profile already exists, full Phase 0 when it doesn't
- Validates output with Pydantic schemas
- Checkpoints progress so a killed run can resume
- Produces `deliverable.xlsx`, `summary.txt`, `methodology.md`, and `execution_log.md`

### Analyst Agent

- Reads the scraper's output and adds analysis on top
- Always: enhanced Excel (Summary / Analysis / Charts sheets), chart PNGs, updated `summary.txt`
- When the data supports a narrative: PDF report with an executive summary and interpreted charts
- When the data has location fields: an interactive geographic map

## Knowledge Base

The knowledge base is the part of Tycho that compounds. Each job writes back what it learned, so the next job on the same site, or using the same pattern, starts from a warmer position than the one before. Everything here lives in the repo as plain markdown and Python, version-controlled alongside the code.

The orchestrator owns all updates and merges them during the post-job retrospective. In-flight agents propose updates in their `execution_log.md` and never touch the shared files directly. That single-writer rule is deliberate: it keeps two jobs on the same domain from stomping on each other.

Four pieces do the work.

### Site Profiles (`skills/sites/`)

One markdown file per domain. Records observed rate limits, auth requirements, anti-bot posture, pagination details, embedded-state paths, approaches that worked, and approaches that failed. A job on the same site six months later starts from everything the first job figured out, including the mistakes it would otherwise repeat.

Each profile carries a viability verdict (green / yellow / red) and a `last_probed` date. The orchestrator uses the verdict to decide whether to take a new job on the domain without spending tokens on fresh recon, and refreshes the profile when the date drifts past 180 days.

The first profile checked in is `sec-gov-edgar` (multi-rate-pool behavior, 10K/query cap, UA-email requirement). New profiles are added one row at a time to `skills/sites/index.md`.

### Toolbox (`skills/toolbox/`)

Reusable code patterns split into focused modules. Current contents:

- `headers.md`: UA profiles, `probe_url()`, session setup.
- `scraping.md`: `RateLimitedSession`, `curl_cffi` TLS impersonation, CSRF forms, `CheckpointScraper`, Playwright wait/scroll, embedded-state extraction, schema validation, circuit breaker.
- `data-cleaning.md`: phone/address/email normalization, price cleaning, URL normalization, fuzzy dedup.
- `output.md`: Excel generation, job directory scaffolding, logging setup.
- `yfinance.md`: Treasury tickers, earnings dates, recommendations pivot, share-class ticker normalization.

The orchestrator promotes a pattern into the toolbox only after it proves useful across two or more jobs and isn't tied to one site. Nothing here is aspirational. Every module earned its place by already solving a concrete problem twice.

### Skill Files (`skills/*.md`)

Methodology, not code. `extraction.md` for scraping techniques, `analysis.md` for chart and PDF construction, `selenium.md` for browser automation, `qc.md` for the quality-control checklist, `voice.md` for the writing style every agent inherits. When a job surfaces a gap in guidance, the retrospective patches the right skill file. Over time these files stop being generic advice and start encoding what actually worked in this repo on real jobs.

### The Retrospective

After every completed job, the orchestrator reads the agent's `execution_log.md` and the scripts it produced, then merges the learnings back:

1. Update the site profile with new rate limits, new gotchas, new approaches that worked or failed. Refresh `last_probed`.
2. Add or update the row in `skills/sites/index.md`.
3. Promote any reusable pattern into the right toolbox module.
4. Patch the relevant skill file if a gap in guidance surfaced.

This is the step that actually makes Tycho smarter. Skipping it leaves the work as a one-off. The orchestrator treats the retrospective as part of the job, not as cleanup, and no job is marked complete without one.

## Extending the Framework

Most extensions fall into one of four shapes:

**Add a site profile.** When a job targets a new domain, the orchestrator runs a researcher first, or writes a profile post-job from the scraper's findings. Start from `skills/sites/template.md`. Add a row to `skills/sites/index.md` the same day; the index is the lookup path for future jobs.

**Add a toolbox module.** When a code pattern proves useful across two or more jobs and isn't tied to one site, promote it into `skills/toolbox/`. Match the file to the kind of pattern: `headers.md` for UA and header work, `scraping.md` for sessions and checkpoints, `data-cleaning.md` for normalization, `output.md` for file generation. Add a one-line entry in `skills/toolbox/README.md` so agents can find it.

**Add a skill file.** Skill files in `skills/*.md` hold methodology; code belongs in `toolbox/`. Add a new skill when a whole class of technique needs explanation (for example, a new browser-automation approach beyond Selenium). Universal rules stay in `base.md`.

**Add a sub-agent.** Uncommon but supported. Drop `{name}.md` into `.claude/agents/` with YAML frontmatter (`name`, `description`, `tools`). Claude Code picks it up as a named subagent type on next start. Keep the prompt focused on one job; don't let agent templates drift into methodology that belongs in `skills/`.

New material written during a job is merged by the orchestrator during the post-job retrospective, not by in-flight agents. That keeps multiple jobs from racing on the same files.

## Job Lifecycle

```
1. You describe a job at the Claude Code prompt
2. Orchestrator assesses feasibility (checks site profile, limitations)
3. Orchestrator scaffolds: jobs/{slug}/, job.json, requirements.md
4. (If new domain) spawn researcher → wait for site profile
5. Spawn scraper agent in the background
6. Agent executes autonomously, checkpointing as it goes
7. Orchestrator runs QC against skills/qc.md
8. (Optional) spawn analyst agent for charts + PDF
9. Orchestrator reports results to you
10. Post-job retrospective: update site profiles, toolbox, skills
```

See [`docs/architecture.md`](docs/architecture.md) for the visual flow with branching paths.

## Examples

Two worked examples, both illustrative (deliverables gitignored):

- [`examples/sp100-earnings-monitor/`](examples/sp100-earnings-monitor/): focused 30-day earnings monitor across the S&P 100. Smaller scope, ~10-minute scrape, two pivots, demonstrates profile reuse and toolbox compounding. Closer to a typical day's job.
- [`examples/sp500-financial-intel/`](examples/sp500-financial-intel/): four-source enrichment across the full S&P 500 (Yahoo Finance, SEC EDGAR, Treasury yields, BLS employment). Larger and longer (~70 minutes), exercises the multi-rate-pool SEC behavior that's now in the site profile.

