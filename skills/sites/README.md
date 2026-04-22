# Site Profiles

Per-domain technical knowledge base. Profiles accumulate what worked and what didn't, so the next job on the same site doesn't start from zero.

## How to Use

**Write ownership:** Only two agents write to this directory: the researcher (creating or refreshing a profile as its primary deliverable) and the orchestrator (merging post-job learnings from a scraper's `execution_log.md`). Scraper and analyst agents never write here directly; they propose updates in their execution log and the orchestrator merges.

**Before probing a new target:** Check if a profile exists for the domain. If it does, read it. You'll know the difficulty, anti-bot status, and working approach instantly.

**When spawning a scraper agent:** Include the relevant site profile content in the agent prompt. Don't make the agent re-discover what you already know.

**After job completion:** Update the profile with new learnings. Rate limits observed at scale, new gotchas, confirmed approaches, failed approaches. If no profile exists, create one from the execution_log. Always refresh `last_probed` to today's date.

**Stale threshold:** a profile is considered stale if `last_probed` is older than 180 days. When an orchestrator is about to spawn a scraper against a stale profile, it should first spawn a researcher to refresh it.

**After probing a new site:** Write a profile even if you decide not to take the job. The recon is worth keeping; it saves the next person a probe.

## Structure

Use `template.md` as the starting point for new profiles.

Each profile has YAML frontmatter:
- `domain`: primary domain
- `aliases`: alternate domains (subdomains, related services)
- `last_probed`: when it was last touched
- `status`: green (working), yellow (works with caveats), red (blocked)
- `protection`: none / basic / moderate / hard / enterprise
- `rendering`: static / api / spa
- `approach`: requests / selenium / curl_cffi / playwright
- `rate_limit_rps`: observed safe rate in requests per second
- `proxy`: none / required / helpful

Sections:
- **Access**: how to authenticate, headers, base URLs
- **Data**: extraction method, endpoints, fields available
- **Pagination**: type and strategy
- **What Doesn't Work**: dead ends
- **Gotchas**: quirks that cost time
- **Rate Limits at Scale**: observed at volume, not just tested

## Index Pattern

Maintain an index.md at this level with a one-line summary per profile:

```
| Domain | Status | Protection | Approach | Notes |
|--------|--------|------------|----------|-------|
| example.gov | green | none | requests | JSON API, 200/query cap |
```

## Examples

See `sec-gov-edgar.md` for a real example profile covering SEC EDGAR, a government data source that works well with simple requests but has specific User-Agent requirements.
