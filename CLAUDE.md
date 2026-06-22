# UtilScan — CLAUDE.md

## What this project is
UtilScan is a personal research tool for US utility holding companies. Given a
ticker, a pipeline pulls public regulatory data and emits structured JSON; a
small web app then lets me **search companies** and view each one's regulatory
overview: a corporate/regulatory structure chart, rate-case tables, and an
AI-generated narrative summary. Personal learning tool, **PUBLIC DATA ONLY**.

## Two-part architecture (read first)
- **Pipeline (offline):** the Python CLI does the heavy lifting (EDGAR, FERC,
  PUC, Claude extraction) and writes the PUBLISHED data as static JSON into
  `web/public/data/` — one `index.json` (company list) plus one
  `{ticker}.json` per company. SQLite remains a LOCAL cache only.
- **Web app (Vercel):** a Next.js app in `web/` that reads those static JSON
  files and renders search + per-company pages. Search is client-side over
  `index.json`. **The web app contains NO secrets and runs NO pipeline code** —
  it only reads committed JSON. `web/public/data/*.json` is the contract between
  the two halves.

Vercel cannot run browser automation, long Python processes, or a persistent
SQLite. Never try to run the pipeline on Vercel.

## Execution environment (cloud vs local)
- **CLOUD-SAFE** (Claude Code on the web): Phase 1 (EDGAR API), Phase 4 (Claude
  API extraction), Phase 5 (emit JSON + build the Next.js app). Only network
  dependency: `sec.gov` / `data.sec.gov`.
- **LOCAL-ONLY** (laptop + real browser): Phase 2 (FERC eLibrary) and Phase 3
  (state PUC adapters). Need `browser-use` + headless browser. Do NOT run these
  in the cloud sandbox; leave documented stubs for local sessions.

## Golden rules (never violate)
- Public data only. Never access paywalled or login-protected content.
- EDGAR requests MUST send a declared `User-Agent` with contact info
  (e.g. `UtilScan/0.1 (your-email@example.com)`), or EDGAR returns 403.
- Never commit secrets. `.env` is gitignored. `ANTHROPIC_API_KEY` is read from
  env by the PIPELINE only — it is never used by the web app and never set on
  Vercel.
- Browser automation must be respectful: obey robots.txt, 2–3s delays, log every
  external request with a timestamp.
- Fail gracefully: one broken source must never kill the whole run.
- Cache aggressively (SQLite). Summarize conservatively. `null` for missing fields.

## Tech stack
- **Pipeline:** Python 3.11+ with `uv`; `httpx` (EDGAR); `browser-use` (FERC/PUC,
  LOCAL only); SQLite single-file cache `utilscan_cache.db`; Anthropic Python SDK
  (Sonnet) for extraction + summarization.
- **Web:** Next.js (App Router) + React + Tailwind, deployed to **Vercel**.
  Reads static JSON from `public/data/`. No backend, no DB.

## Project layout
- `pipeline/` — Python package: `cli.py`, `config.py`, `db.py`, `edgar.py`,
  `ferc.py`, `extract.py`, `emit.py` (writes JSON to `../web/public/data/`),
  `puc/` (one adapter per state in a `STATE_ADAPTERS` registry).
- `web/` — Next.js app. `public/data/` holds `index.json` + `{ticker}.json`.
  Pages: `/` (search + list), `/company/[ticker]` (overview).
- `pipeline/cache/`, `pipeline/output/` — gitignored runtime dirs (SQLite lives here).
- `tests/` — pytest with fixtures.

## Build order (strict — one ROADMAP item per session, checkpoint each)
See `ROADMAP.md`. After each item: run tests, tick its box, open a PR on a
`claude/` branch. **Never push to `main`.**

## CLI
`utilscan TICKER [--state XX] [--refresh] [--no-browser] [--report-only]`
`utilscan --list`  (--report-only / emit re-writes the JSON for the web app)

## Definition of done per phase
- **Phase 1:** `utilscan EXC --no-browser` prints a correct corporate tree and
  writes `companies`/`subsidiaries` rows. Validate on EXC, SO, NEE, DUK, D.
- **Phase 4:** extraction prompt returns valid JSON for a sample order PDF in
  `tests/fixtures/`.
- **Phase 5:** pipeline writes `web/public/data/EXC.json` + updates `index.json`;
  the Next.js app renders the search page and the EXC company page locally
  (`web/` runs with `npm run dev`).
- **Deploy:** pushing to `main` auto-deploys `web/` to Vercel; the live site
  searches and shows the seeded companies.
- **Phases 2/3 (local):** dockets per company/state printed at a checkpoint
  before any download/summarization.
