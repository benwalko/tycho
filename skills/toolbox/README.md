# Toolbox -- Code Recipes

Reference library for scraper agents. Recipes are split into focused modules. Read only what you need.

| Module | Path | Contents |
|--------|------|----------|
| **Headers & UA Probing** | `headers.md` | UA profiles, `probe_url()`, `get_session_with_profile()`, common patterns |
| **Scraping Patterns** | `scraping.md` | `RateLimitedSession`, `curl_cffi` TLS impersonation, CSRF forms, `CheckpointScraper`, Playwright wait/scroll, embedded state extraction, schema validation, circuit breaker |
| **Data Cleaning** | `data-cleaning.md` | Phone normalization, address standardization, email validation, price cleaning, URL normalization, `validate_data()`, fuzzy dedup, pandas cleaning |
| **Output Generation** | `output.md` | `create_xlsx()`, `df_to_xlsx()`, `csv_to_xlsx()`, `init_job_dir()`, `setup_logging()` |
| **yfinance Patterns** | `yfinance.md` | Treasury tickers, earnings dates, revenue estimate/actual patterns, recommendations pivot, sector taxonomy mapping, share-class ticker normalization |

## Browser Automation

See `skills/selenium.md` for the full Selenium skill: setup, anti-detection, API discovery (HAR capture), cookie transfer, form interaction, pagination, and troubleshooting.
