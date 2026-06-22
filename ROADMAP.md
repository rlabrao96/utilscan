# UtilScan — ROADMAP

Work top to bottom. Do **ONE unchecked item per session**. After completing an
item: run tests, tick the box, open a PR on a `claude/` branch.
`[CLOUD]` = safe for Claude Code on the web. `[LOCAL]` = laptop + `browser-use`.

## Phase 0 — Scaffolding  [CLOUD]
- [ ] `uv` project in `pipeline/`; package skeleton (`cli.py`, `config.py`,
      `db.py`); `pyproject.toml` with httpx, anthropic; pytest configured.
- [ ] `config.py` reads `.env` (ANTHROPIC_API_KEY, CACHE_DIR, OUTPUT_DIR,
      BROWSER_HEADLESS). Confirm `.gitignore` covers `.env`, `pipeline/cache/`,
      `pipeline/output/`, `*.db`, `.venv`, `web/node_modules`, `web/.next`,
      `.vercel`.
- [ ] `db.py`: create SQLite schema (companies, subsidiaries, rate_cases,
      documents) idempotently.

## Phase 1 — EDGAR layer  [CLOUD]
- [ ] `edgar.py`: ticker→CIK + fetch latest 10-K via `data.sec.gov` submissions
      API. Declared `User-Agent`. httpx with retries + polite rate limit.
- [ ] Extraction: feed 10-K text to Claude (Sonnet) → corporate-tree JSON
      (holdco; regulated subs w/ state, regulator, type, FERC flag; unregulated
      subs; earnings mix if disclosed).
- [ ] Persist tree to `companies` + `subsidiaries`. `utilscan EXC --no-browser`
      prints it. **CHECKPOINT.**
- [ ] Validate on SO, NEE, DUK, D. Tune the prompt until all 5 look right.

## Phase 4 (partial) — Extraction prompt  [CLOUD]
- [ ] `extract.py`: rate-case extraction prompt (JSON schema from the spec).
      Run on a sample order PDF in `tests/fixtures/`. Assert valid JSON.

## Phase 5 — Data emit + web app  [CLOUD]
- [ ] `emit.py`: from cache, write `web/public/data/{TICKER}.json` (full company
      payload: tree, subsidiaries, rate_cases, narrative) and update
      `web/public/data/index.json` (list: ticker, name, # subs, # cases).
      `utilscan EXC --report-only` regenerates these. **CHECKPOINT:** inspect JSON.
- [ ] Scaffold the Next.js app in `web/` (App Router + Tailwind). Use the
      **frontend-design skill** for the UI. Dark navy / white / accent,
      pitch-book feel.
- [ ] Home page `/`: search box + company list, client-side filter over
      `index.json`.
- [ ] Company page `/company/[ticker]`: Section A structure tree (SVG/CSS,
      sentiment color-coding), Section B rate-case tables + rate-base & ROE
      charts, Section C narrative. Reads `{ticker}.json`. **CHECKPOINT:** review
      EXC page at `npm run dev`.

## Phase 5b — Deploy to Vercel  [MANUAL, you do this]
- [ ] Import the GitHub repo in Vercel, set **Root Directory = `web`**,
      framework auto-detected (Next.js). No env vars needed (app is read-only).
- [ ] Confirm auto-deploy on push to `main`; live site searches + shows seeded
      companies.

## Phase 2 — FERC layer  [LOCAL]
- [ ] `ferc.py`: `browser-use` on elibrary.ferc.gov — search holdco + each sub,
      filter ER/EL/FA dockets, grab docket/date/description/status. Cache to
      `rate_cases` (jurisdiction = "FERC"). **CHECKPOINT:** print dockets first.

## Phase 3 — State PUC adapters  [LOCAL]
- [ ] `puc/registry.py` + base adapter interface + `STATE_ADAPTERS` dict.
- [ ] Before any adapter, check if the PUC has an API / open-data portal; prefer
      it over `browser-use`.
- [ ] IL adapter (ICC) — ComEd test case. **CHECKPOINT:** print dockets.
- [ ] VA adapter (SCC) — Dominion.
- [ ] NC adapter (NCUC) — Duke Energy Carolinas.
- [ ] CA adapter (CPUC).
- [ ] (stretch) NY, TX, FL, PA, OH, GA.

## Phase 6 — Integrate & polish  [MIXED]
- [ ] Wire Phase 4 extraction over every docket from Phases 2–3; populate
      `rate_cases`.
- [ ] Re-run `emit.py` so the web app shows full rate-case data; push → Vercel
      redeploys.
- [ ] CLI flags, structured logging, graceful per-source error handling.
- [ ] End-to-end: `utilscan EXC` then view live on Vercel. Ship.
