# Failure Report Template

When an agent hits a hard blocker it cannot work around, it must stop and report using this format. This gives the orchestrator structured info to decide whether to retry with a different approach, escalate, or abandon.

## Where the report goes

The agent must put the failure report in BOTH places:

1. **Return it as the agent's final response** so the orchestrator sees it immediately in the completion notification.
2. **Append it to `execution_log.md`** under a `## Failure Report` heading so the post-job retrospective can read it even if the orchestrator session has moved on.

Also update `job.json`: set `status` to `failed`, add `failed_at` and `failure_reason` (a one-line summary; the full report lives in the two places above).

## Format

```markdown
## Status: BLOCKED

**What was attempted:**
- Specific approaches tried, in order
- With numbers: "Selenium with chrome_win UA, rate 1.5s, 200 req before first block"
- Not vague: avoid "tried several approaches"

**Root cause:**
- The specific thing that blocked progress
- E.g. "Cloudflare Turnstile challenge on every request, cannot be bypassed with cloudscraper"
- E.g. "API returns 403 for all non-browser requests; JA3 fingerprint rejection"

**Data collected so far:**
- Partial results count (if any)
- Path to partial output file
- What fields were captured

**Recommendation:**
- Alternative approach to try (e.g. "check mobile site, often less protected")
- Or: "abandon, site is unscrapable without residential proxies"
- Or: "escalate to operator for credentials"
```

## When to Use

Use this when:
- All 3 different approaches in the escalation path have failed
- A hard wall is hit (login, CAPTCHA, IP ban, ToS violation)
- Scope is 10x larger than described

Don't use this when:
- You just haven't tried enough things yet (keep trying)
- A single approach failed (pivot)
- You're tired (keep going)

## After Reporting

The orchestrator decides:
- **Retry with new approach**: if the agent missed something obvious
- **Escalate**: the operator might have credentials, domain knowledge, or a different angle
- **Abandon**: some sites just aren't worth the effort

The agent does NOT retry on its own after reporting BLOCKED.
