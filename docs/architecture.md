# Architecture

## The Three-Layer Model

```
Operator (you)
   ↓ directs
Orchestrator (Claude Code main session)
   ↓ spawns
Sub-agents (researcher / scraper / analyst)
   ↓ produce
Deliverables (Excel, PDF, charts)
```

Each layer has a clear role:

- **Operator** sets intent and reviews output.
- **Orchestrator** holds context, makes decisions, runs QC, never does the work itself.
- **Sub-agents** execute autonomously within their narrow scope.

## Why Multi-Agent

One agent doing everything sounds simpler. It isn't:

1. **Context budget.** A scraper that loads the full orchestration context burns through tokens. A scraper that reads only its own template plus the requirements stays lean.
2. **Parallelism.** The orchestrator can run three jobs at once and none of them blocks the others.
3. **Specialization.** The scraper prompt is tuned for recon discipline and execution speed. The analyst prompt is tuned for narrative writing and chart aesthetics. Folding them into one prompt dilutes both.
4. **Failure isolation.** A failing scraper returns a structured blocker. The orchestrator decides what's next: retry with different context, escalate, or drop the job. The main session's state stays clean either way.

## Agent Types

### Researcher Agent

- Template: `.claude/agents/researcher.md`
- Tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch
- Background: yes
- Turns: up to 30
- Primary output: a site profile at `skills/sites/{domain}.md` and an index update

Recon-only. Probes an unknown target with multiple UA profiles, identifies anti-bot and rendering, looks for embedded JSON or APIs, checks pagination. Returns a viability verdict (green/yellow/red) and writes a reusable profile. Used before accepting a job on an unknown site, or to refresh a stale profile.

### Scraper Agent

- Template: `.claude/agents/scraper.md`
- Tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch
- Background: yes
- Turns: up to 100
- Primary output: `deliverable.xlsx`

Follows a recon-first methodology: Phase 0 (discovery) → Phase 1 (simplest approach) → Phase 2 (extraction with validation) → Phase 3 (self-check). Never jumps straight to Selenium when requests might work.

### Analyst Agent

- Template: `.claude/agents/analyst.md`
- Tools: Bash, Read, Write, Edit, Glob, Grep
- Background: yes
- Turns: up to 50
- Primary output: enhanced `deliverable.xlsx` + optional `analysis_report.pdf` + charts

Reads what the scraper produced. Doesn't re-scrape. Produces a consistent analysis package: enhanced Excel, chart PNGs, and (when the data supports it) a PDF report and geographic map.

## Agent Invocation

All three agents are loaded as named subagent types by Claude Code. The orchestrator spawns them via the Agent tool with `subagent_type: <name>` and `run_in_background: true`. Job-specific context (directory paths, site profile contents, requirement references) goes in the prompt string. The template file itself is not injected; Claude Code applies it to the subagent automatically.

## The Job Lifecycle

```
┌────────────────┐
│ Request arrives│
└────────┬───────┘
         │
         ▼
┌─────────────────────┐
│ Feasibility assess  │  (orchestrator uses skills/extraction.md,
│                     │   site profiles, base.md)
└────────┬────────────┘
         │
         ├── Not viable → report to operator
         │
         ▼
┌─────────────────────┐
│ Scaffold job        │  (create dir, job.json, requirements.md)
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Spawn scraper agent │  (background, with full context)
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Agent runs          │  (recon → extract → validate → deliverables)
└────────┬────────────┘
         │
         ├── BLOCKED → report to operator
         │
         ▼
┌─────────────────────┐
│ QC                  │  (skills/qc.md checklist)
└────────┬────────────┘
         │
         ├── Issues → fix directly or spawn follow-up agent
         │
         ▼
┌─────────────────────┐
│ (Optional) Analysis │  (spawn analyst for enhanced Excel + charts + PDF)
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Report to operator  │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Retrospective       │  (update site profiles, toolbox, skills)
└─────────────────────┘
```

## State and Persistence

Jobs live on disk, not in agent memory. Any agent can pick up a job mid-stream by reading:

- `requirements.md`: what to do
- `job.json`: current status and stats
- `execution_log.md`: what has been tried so far
- `output/*`: what has been produced

In practice that buys you three things:

- A killed agent can be restarted without losing progress, as long as it was checkpointing to disk.
- The orchestrator inspects job state by reading files, not by holding it in memory.
- An orchestrator session a week from now can pick up where this one left off.

## Where Knowledge Lives

Three layers:

1. **`skills/*.md`**: operational playbooks. How the work is done. Changes rarely.
2. **`skills/sites/*.md`**: per-domain facts. Updated after every job on that domain.
3. **`skills/toolbox/*.md`**: reusable code patterns. Promoted when a technique proves useful across two or more jobs.

Every completed job should leave the knowledge base slightly better than it found it.

## What This Framework Isn't

- Not a scheduler. Jobs run when the orchestrator spawns them, not on a cron.
- Not a queue. If multiple jobs arrive, the orchestrator decides what runs in parallel.
- Not a hosted service. The whole thing runs inside a Claude Code session on your machine.
- Not a replacement for human judgment. The operator still decides what to scrape and what to do with the result.
