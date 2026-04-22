# Voice

The writing voice for this project. Applies to:

- Anything an agent writes for a human reader: `methodology.md`, `summary.txt`, `execution_log.md`, research reports, site profiles, PDF narratives.
- Framework documentation itself: README, CLAUDE.md, skills, site profiles, toolbox entries, this file.

The whole project should read like one person wrote it.

## Tone

Professional but not stiff. Confident, not hedgy. Direct, not clipped. Think of a senior engineer writing to another senior engineer about work they both care about. Short sentences are fine; mix short and medium sentences so the rhythm stays readable.

The shorthand: professionally optimistic. Confident in what the framework does, honest about what it doesn't, no sales gloss.

## Banned vocabulary

These read as corporate filler or AI boilerplate. Cut them. If a sentence needs one of these to make sense, the sentence is probably hollow.

- **Verbs:** `leverage`, `utilize`, `facilitate`, `foster`, `harness`, `empower`, `unlock`, `embark`, `navigate` (the complexities of), `delve`, `streamline`
- **Adjectives:** `robust`, `comprehensive`, `seamless`, `cutting-edge`, `state-of-the-art`, `game-changing`, `transformative`, `revolutionary`, `meticulous`, `pivotal`, `paramount`, `crucial`, `vital`, `elegant`, `powerful` (when unquantified)
- **Nouns / metaphors:** `tapestry`, `testament`, `realm`, `journey` (as metaphor), `landscape` (as metaphor), `ecosystem` (unless literal), `deep dive`, `game-changer`, `one-stop shop`
- **Transitions:** `Furthermore`, `Additionally`, `Moreover`, `In conclusion`, `It's worth noting`, `At the end of the day`, `When it comes to`, `In today's fast-paced...`, `In the world of...`

## Patterns to avoid

- **Em-dashes.** Use a period, comma, semicolon, or colon instead.
- **Triadic parallel structure.** Sentences like "fast, reliable, and scalable" or "scraping, enriching, and delivering" stack abstractions. Pick the one that matters and say why.
- **Empty intensifiers.** `very`, `really`, `truly`, `quite`, `absolutely`, `significantly` usually add nothing. Cut them or replace with a specific number.
- **Hedged confidence.** `may potentially`, `can sometimes`, `could possibly` muddies the claim. Either it does or it doesn't. Hedge only when the uncertainty is real, and name what the uncertainty is.
- **Abstract nouns when a verb will do.** "Provides facilitation of scraping" vs. "Runs scraping jobs." Prefer the verb.
- **Warmup sentences.** If the first sentence of a section restates the section title, delete it.
- **Over-explaining the obvious.** If the reader can figure it out from context, don't restate it.
- **Name-the-thing-twice.** "The orchestrator, which coordinates agents," followed two paragraphs later by "The orchestrator, the coordinator of agents," is AI-flavored repetition. Name it once, then just "the orchestrator."

## Things to do

- **Numbers over adjectives.** "200 results in 3.2s" beats "returned results quickly." "Blocked after 47 requests at 1 req/s" beats "rate limiting was moderate."
- **Active voice by default.** "The orchestrator spawns the scraper" not "The scraper is spawned."
- **Lead with the point.** The first sentence of a paragraph should carry the main claim. Supporting detail follows.
- **Name things directly.** "Scraper agent" not "autonomous execution entity." "Site profile" not "per-domain technical knowledge artifact."
- **Trust the reader.** If you've established context, don't restate it every paragraph.
- **Be honest about scope and tradeoffs.** If a feature works only under certain conditions, say so plainly. Professional optimism is not the same as overclaiming.
- **One voice.** Even when a doc has sections by different agents, the writing should feel like one person wrote it.

## Quick test

If a sentence could appear verbatim in a SaaS marketing page, rewrite it. If it could appear in a sober engineering doc (Stripe, the Google SRE book, a well-written RFC), you're on target.

## Before and after

Concrete examples of the same idea, AI-flavored vs. on-voice:

**On a framework's purpose.**

> Comprehensive, robust scraping framework that leverages multi-agent architecture to seamlessly extract and enrich data across diverse sources.

becomes

> A scraping framework that runs end to end inside a Claude Code session. Multi-source enrichment is the sweet spot.

**On agent behavior.**

> The orchestrator empowers users to navigate the complexities of modern web scraping by harnessing the power of specialized agents.

becomes

> The orchestrator runs the job, spawns agents, runs QC, updates the knowledge base.

**On rate-limit findings.**

> Furthermore, the API exhibited robust rate-limiting characteristics that necessitated careful pacing strategies.

becomes

> efts.sec.gov: 5 req/s sustained across 10,000+ requests, zero 429s. www.sec.gov: 3-4 req/s safer; multi-minute cooldowns trigger easily if exceeded.

**On data quality notes.**

> The dataset comprehensively captures key financial metrics across the full S&P 500 universe with robust coverage of fundamental indicators.

becomes

> 503 rows, 35 columns. 100% coverage on 21 fields, >90% on 28 of 35. Insider sentiment at 14.7% (SEC rate-limit constraint, explained in summary.txt).

**Pattern across all four:** strip the abstraction, name the thing, lead with numbers when you have them.

## Where this gets enforced

- Agents read this file before producing prose; pointers live in `skills/base.md` and each agent template.
- The QC checklist (`skills/qc.md`) checks output against this file before the orchestrator reports results.
- When you spot a violation in this repo, fix it directly. The voice isn't aspirational; it's what the project actually reads like.
