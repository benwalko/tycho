# Site Registry

Per-domain knowledge base. Check here before probing a new target or spawning a scraper. The list grows as the researcher writes new profiles after each fresh probe.

## Active Sites

| Domain | Status | Protection | Approach | Notes |
|--------|--------|------------|----------|-------|
| [sec.gov/edgar](sec-gov-edgar.md) | green | basic | requests | Requires email in UA, 10K/query cap, multi-rate-pool |

## How to Use

**During feasibility assessment:** Check this index. If domain is listed, pull the profile for instant difficulty assessment.

**During agent spawn:** Include the relevant site profile content in the agent prompt.

**After job completion:** Update the site profile with new learnings (rate limits at scale, new gotchas).

**After probing:** Write a new site profile instead of a throwaway report.

## Starting a New Profile

Use `template.md` as the starting point. Fill in whatever you learned during recon. Even a partially-filled profile is more useful than no profile.
