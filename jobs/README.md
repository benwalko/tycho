# Jobs

Active job directories live here. Each job gets its own folder with a slug derived from the project name.

## Structure

```
jobs/
  my-job-slug/
    job.json                  # metadata: title, status, timestamps, stats
    requirements.md           # what to scrape, what fields, what format
    execution_log.md          # scraper's internal log of approach and discoveries
    execution_log_analysis.md # analyst's log (if analyst ran)
    scripts/                  # scraper's Python scripts. Preserved for retrospective.
    scripts_analysis/         # analyst's Python scripts (if analyst ran). Preserved for retrospective.
    output/
      deliverable.xlsx        # primary deliverable
      summary.txt             # coverage stats, row counts, field completeness
      methodology.md          # how the work was executed (plain-language writeup)
      charts/                 # chart PNGs (if analyst ran)
      analysis_report.pdf     # narrative report (if analyst ran and data warranted it)
```

All `scripts*/` and `output/charts/` content is gitignored by `jobs/*/` but preserved on disk for the post-job retrospective.

## Job Statuses

- `created` - scaffolded but not yet started
- `active` - agent running
- `completed` - agent finished, deliverables produced
- `failed` - agent blocked, see execution_log for details

## `job.json` Schema

Canonical spec for the metadata file. Every scraper agent reads and updates this; the orchestrator reads it during QC and for status reporting.

### Fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `title` | string | yes | Human-readable job title. |
| `slug` | string | yes | Directory slug, matches the folder name. |
| `status` | enum | yes | One of: `created`, `active`, `completed`, `failed`. |
| `created` | ISO-8601 date | yes | When the orchestrator scaffolded the job. |
| `started_at` | ISO-8601 datetime | no | When the scraper agent began execution. |
| `completed_at` | ISO-8601 datetime | on completion | Required when status is `completed`. |
| `failed_at` | ISO-8601 datetime | on failure | Required when status is `failed`. |
| `failure_reason` | string | on failure | Short reason from the agent's BLOCKED report. |
| `target_domain` | string | no | Primary domain, for cross-reference with `skills/sites/`. |
| `records` | object | on completion | See below. |

`records` object (present when status is `completed`):

| Field | Type | Notes |
|-------|------|-------|
| `total` | int | Row count in `deliverable.xlsx`. |
| `coverage` | string | Overall field-coverage summary, e.g. `"90%+ on 28 of 35 fields"`. |

### State transitions — who writes what

| Transition | Written by | Fields set |
|------------|-----------|------------|
| Scaffold | Orchestrator | `title`, `slug`, `status: created`, `created`, `target_domain` (if known) |
| Agent starts | Scraper agent (on first action) | `status: active`, `started_at` |
| Job succeeds | Scraper agent (before returning) | `status: completed`, `completed_at`, `records` |
| Job blocked | Scraper agent (before returning) | `status: failed`, `failed_at`, `failure_reason` |

The orchestrator never changes `status` to `active`, `completed`, or `failed` directly. Agents own their own lifecycle updates so the file stays consistent even when the orchestrator session is doing other work.

### Example

```json
{
  "title": "Scrape all dealer locations from example.com",
  "slug": "example-dealer-locations",
  "status": "completed",
  "target_domain": "example.com",
  "created": "2026-01-15",
  "started_at": "2026-01-15T10:22:04Z",
  "completed_at": "2026-01-15T10:47:18Z",
  "records": {
    "total": 79,
    "coverage": "100%"
  }
}
```

### Pydantic Model

Agents that want runtime validation can import this model:

```python
from datetime import datetime
from typing import Literal, Optional
from pydantic import BaseModel

class JobRecords(BaseModel):
    total: int
    coverage: str

class JobMeta(BaseModel):
    title: str
    slug: str
    status: Literal["created", "active", "completed", "failed"]
    created: str  # ISO-8601 date
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    failed_at: Optional[datetime] = None
    failure_reason: Optional[str] = None
    target_domain: Optional[str] = None
    records: Optional[JobRecords] = None
```

Write a job.json:

```python
import json
with open("jobs/my-slug/job.json", "w") as f:
    json.dump(JobMeta(...).model_dump(exclude_none=True), f, indent=2)
```

Read and update:

```python
with open(path) as f:
    meta = JobMeta.model_validate_json(f.read())
meta.status = "completed"
meta.completed_at = datetime.now(timezone.utc)
with open(path, "w") as f:
    json.dump(meta.model_dump(exclude_none=True, mode="json"), f, indent=2)
```

## Revisions

When a job needs a revision after completion:
1. Create a `revisions/rev-N/` directory
2. Copy current `output/` into it as a snapshot
3. Spawn a new agent with revision instructions
4. Agent overwrites `output/` with the updated deliverable
5. Operator can diff the snapshots to see what changed

Do NOT pre-create a `revisions/` directory. Create it on-demand only when needed.

## Worked Examples

See `examples/` in the repo root for completed jobs with full write-ups.
