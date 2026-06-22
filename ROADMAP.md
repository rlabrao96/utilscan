# UtilScan — ROADMAP

Work top to bottom. Do **ONE unchecked item per session**. After completing an
item: run tests, tick the box, open a PR on a `claude/` branch.
`[CLOUD]` = safe for Claude Code on the web. `[LOCAL]` = run on a laptop with
`browser-use`.

## Phase 0 — Scaffolding  [CLOUD]
- [ ] `uv` project init; package skeleton (`utilscan/` with `cli.py`,
      `config.py`, `db.py`); `pyproject.toml` with httpx, anthropic, jinja2;
      pytest configured.
- [ ] `config.py` reads `.env` (ANTHROPIC_API_KEY, CACHE_DIR, OUTPUT_DIR,
      BROWSER_HEADLESS). `.gitignore` covers `.env`, `cache/`, `output/`,
      `*.db`, `.venv`.
- [ ] `db.py`: create the SQLite schema (companies, subsidiaries, rate_cases,
      documents) idempotently on first run.

## Phase 1 — EDGAR layer  [CLOUD]
- [ ] `edgar.py`: ticker→CIK resolution + fetch latest 10-K via the
      `data.sec.gov` submissions API. Declared `User-Agent` header. httpx with
      retries + polite rate limit.
- [ ] Extraction: feed the 10-K text to Claude (Sonnet) to produce the
      corporate-tree JSON (holdco; regulated subs w/ state, regulator, type,
      FERC flag; unregulated subs; earnings mix if disclosed).
- [ ] Persist the tree to `companies` + `subsidiaries`. `utilscan EXC
      --no-browser` prints the tree. **CHECKPOINT.**
- [ ] Validate the tree on SO, NEE, DUK, D. Tune the extraction prompt until all
      5 look right.

## Phase 4 (partial) — Extraction prompt  [CLOUD]
- [ ] `extract.py`: the rate-case extraction prompt (the JSON schema from the
      spec). Run it on a sample order PDF in `tests/fixtures/` (manually added).
      Assert valid JSON, fields populated.

## Phase 5 — Report generation  [CLOUD]
- [ ] `report.py`: self-contained HTML — Section A (SVG/CSS structure tree with
      sentiment color-coding), Section B (master + per-sub tables, rate-base &
      ROE charts), Section C (narrative via Claude, 400–600 words).
- [ ] `utilscan EXC --report-only` writes
      `output/EXC_regulatory_overview.html` + `EXC_data.json` from cache.
      **CHECKPOINT:** open the HTML, review.

## Phase 2 — FERC layer  [LOCAL]
- [ ] `ferc.py`: `browser-use` against elibrary.ferc.gov — search holdco + each
      sub, filter ER/EL/FA dockets, grab docket/date/description/status. Cache to
      `rate_cases` (jurisdiction = "FERC"). **CHECKPOINT:** print dockets before
      summarizing.

## Phase 3 — State PUC adapters  [LOCAL]
- [ ] `puc/registry.py` + base adapter interface + `STATE_ADAPTERS` dict.
- [ ] Before writing any adapter, check whether the state PUC exposes an
      API/open-data portal; prefer it over `browser-use`.
- [ ] IL adapter (ICC) — ComEd as test case. **CHECKPOINT:** print dockets found.
- [ ] VA adapter (SCC) — Dominion.
- [ ] NC adapter (NCUC) — Duke Energy Carolinas.
- [ ] CA adapter (CPUC).
- [ ] (stretch) NY, TX, FL, PA, OH, GA.

## Phase 6 — Integrate & polish  [MIXED]
- [ ] Wire Phase 4 extraction over every docket found in Phases 2–3; populate
      `rate_cases` fully.
- [ ] Regenerate reports with full data.
- [ ] CLI flags (--refresh, --state, --report-only, --list), structured logging,
      graceful per-source error handling.
- [ ] End-to-end run: `utilscan EXC`. Ship.
