# Limitations

What this framework can't do, plus the real tradeoffs of what it can. Read this before pitching a job to the orchestrator that touches a known-hard target.

## Hard blockers (don't attempt)

- **Cloudflare Turnstile** or any managed challenge ("Verify you are human"). Requires a real browser and a human signal. `cloudscraper` only handles older basic-JS challenges.
- **Enterprise anti-bot:** Akamai Bot Manager, PerimeterX, DataDome. Same constraint; these need residential proxies and headless-browser fingerprinting that aren't built in.
- **Login walls without credentials.** No interactive auth, no credential storage. If the data needs an account, you have to provide creds and add a custom auth flow yourself.
- **CAPTCHAs.** No solver integrations included.
- **Sites with explicit scraping prohibitions in their ToS.** LinkedIn, Yelp, Facebook, Instagram, ZoomInfo, Apollo. Off-limits regardless of technical feasibility.

## Works, but with care

- **Commercial e-commerce** (Amazon, Best Buy, Walmart, Target, Zillow). Possible at small scale with conservative rate limits, but a single IP will get noticed. Sustained scraping needs proxy rotation, which isn't built in.
- **JavaScript-heavy SPAs with no underlying API.** Selenium handles them, but throughput drops 10x to 50x compared to direct HTTP.
- **Volume against well-protected sites.** 5,000 records from a government API is a non-event. 500 records from a protected commercial site is hard.

## Works well

- **Government data sources** (`.gov`, public registries, EDGAR, BLS, FRED, Treasury Direct). Clean APIs, generous rate limits, no anti-bot to speak of.
- **Public APIs** with documented endpoints.
- **Static HTML directories** with predictable pagination.
- **Sitemap-based crawls** when the sitemap is published.
- **Multi-source enrichment.** Combining two to four public sources into one deliverable. This is the framework's sweet spot.

## Volume guidance

| Records | Where it works |
|---------|---------------|
| 100 - 1,000 | Most sites, fast. |
| 1,000 - 10,000 | Government and public APIs handle this comfortably. Commercial sites need checkpointing and patience. |
| 10,000 - 100,000 | Realistic only on truly open APIs (SEC EDGAR, FRED). Use the staged-execution pattern in `skills/extraction.md`. |
| 100,000+ | Possible but requires planning, multi-day runs, and careful rate budgeting. Not the framework's primary use case. |

## When to reach for something else

- **Scheduled or cron-based scraping.** This framework runs jobs on demand inside a Claude Code session. Continuous monitoring needs an external scheduler.
- **Hosted or multi-tenant.** Everything runs on the operator's machine. No multi-user setup.
- **Live feeds.** Output is snapshots, not streams.
- **Real-time dashboards.** Output is files (XLSX, PDF, PNG), not a serving layer.

## Asking the orchestrator about feasibility

If you're not sure whether a target is in scope, ask the orchestrator before scaffolding a job:

> Is yelp.com viable for scraping with this framework?

It will check the existing site profiles, the limits in this doc, and reply with a verdict. You can also have it spawn a researcher to probe an unknown site and write a profile, even if you don't end up taking the job.
