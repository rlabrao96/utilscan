# UtilScan — CLAUDE.md

## What this project is
UtilScan is a personal research tool for US utility holding companies. Given a
ticker, it produces a regulatory overview: a corporate/regulatory structure
chart, rate-case tables, and an AI-generated narrative summary. Built for
personal learning and market orientation in power & energy transition, using
**PUBLIC DATA ONLY**.

## Golden rules (never violate)
- Public data only. Never access paywalled or login-protected content.
- EDGAR requests MUST send a declared `User-Agent` header with contact info
  (e.g. `UtilScan/0.1 (your-email@example.com)`). Without it, EDGAR returns 403.
- Never commit secrets. `.env` is gitignored. Read `ANTHROPIC_API_KEY` from the
  environment — never hardcode it.
- Browser automation must be respectful: obey robots.txt, add 2–3s delays
  between requests, and log every external request with a timestamp.
- Fail gracefully: one broken source must never kill the whole run. Catch, log,
  continue.
- Cache aggressively (SQLite). Summarize conservatively. Return `null` for any
  field not present in the source.

## Execution environment (IMPORTANT — read before coding)
This repo is built in two halves:
- **CLOUD-SAFE** (Claude Code on the web): Phase 1 (EDGAR API), Phase 4 (Claude
  API extraction), Phase 5 (HTML report). Pure API + compute, no browser. The
  ONLY external network dependency is `sec.gov` / `data.sec.gov`.
- **LOCAL-ONLY** (run on a laptop with a real browser): Phase 2 (FERC eLibrary)
  and Phase 3 (state PUC adapters). These need `browser-use` + a headless
  browser and broad network egress. Do **NOT** attempt the browser phases in the
  cloud sandbox.

When working in a cloud session, only implement cloud-safe phases. Leave browser
phases as documented stubs (clear TODO + a defined interface) for local sessions
to fill in.

## Tech stack
- Python 3.11+, managed with `uv`.
- `httpx` for direct API calls (EDGAR).
- `browser-use` for browser automation (FERC, PUC) — **LOCAL ONLY**.
- SQLite single-file cache: `utilscan_cache.db`.
- Anthropic Python SDK for Claude API (extraction + summarization). Use **Sonnet**.
- `jinja2` or f-strings for HTML. No heavy frameworks. Keep it lean.

## Project layout
- `utilscan/` — package: `cli.py`, `config.py`, `db.py`, `edgar.py`, `ferc.py`,
  `extract.py`, `report.py`, `puc/`
- `utilscan/puc/` — one adapter module per state, registered in a
  `STATE_ADAPTERS` dict (abbrev → adapter fn). Skip cleanly if no adapter exists:
  log `No adapter for {STATE}`.
- `cache/`, `output/` — gitignored runtime dirs.
- `tests/` — pytest, with sample fixtures in `tests/fixtures/`.

## Build order (strict — finish and checkpoint each before the next)
See `ROADMAP.md`. Do **exactly one** ROADMAP item per session unless told
otherwise. After each item: run tests, tick its checkbox in `ROADMAP.md`, and
open a PR on a `claude/` branch. **Never push to `main`.**

## CLI
`utilscan TICKER [--state XX] [--refresh] [--no-browser] [--report-only]`
`utilscan --list`

## Definition of done per phase
- **Phase 1:** `utilscan EXC --no-browser` prints a correct corporate tree and
  writes `companies` / `subsidiaries` rows to SQLite. Validate on EXC, SO, NEE,
  DUK, D.
- **Phase 4:** the extraction prompt returns valid JSON for a sample rate-order
  PDF placed in `tests/fixtures/`.
- **Phase 5:** `utilscan EXC --report-only` generates
  `output/EXC_regulatory_overview.html` and `output/EXC_data.json` from cache.
- **Phases 2/3 (local):** dockets found per company/state are printed at a
  checkpoint **before** any download or summarization.
