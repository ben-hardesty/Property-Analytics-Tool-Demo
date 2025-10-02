# Property Analytics Tool (PAT) — README

NOTE: This is a minimal "**case-study**" copy of the main project. The **full codebase is private** and available on request.

---

## Overview

**PAT** is a CLI-driven, SQLite-backed toolkit with an importable Python package for UK property analysis. It wraps the PropertyData API with **single-source parameter validation**, runs **daily cohort pulls** , and persists **raw JSON verbatim** alongside light denormalised fields for analytics.

- **CLI-first, importable** — run from the terminal or import `pat` in notebooks.
- **Single source of truth** — enums & bounds live in constants.py; thin wrappers in api/propertydata_api.py.
- **Cohort snapshots** — daily pulls for curated postcode sets.
- **Canonical storage** — JSON always; SQLite as the authoritative store with postcode-aware, stable dedupe.
- **Analytics-ready** — minimal denorm helpers and SQL views for quick time-series joins.
- **Discoverable** — structured endpoint grouping (6 categories) and a credit estimator.
- **Automated ops** — daily ETL and weekly DB backups via GitHub Actions.

### How It Works

- **Cohort-driven fetches** — Postcodes live in `config/cohorts.yaml` and sync to `dim_cohort*`; the daily ETL loops through them.
- **Postcode-aware dedupe** — Rows are unique on `(endpoint_name, context_postcode, checksum)` where the checksum is a **stable projection** that ignores volatile fields (e.g., `process_time`, `url`).
- **DB-driven ETL** — `api_responses` stores canonical JSON + request/context metadata; denormalised helpers (e.g., `effective_radius_mi`, `points_requested`) are populated for fast slicing.
- **Built-in views** — Ready-to-query SQL views for the current focus endpoints:
  - `v_prices`, `v_rents`, `v_sold_prices`, `v_crime`
- **Automation included** — GitHub Actions run the **Daily ETL** (updates a rolling `db-latest` release) and a **Weekly backup** (dated zip artifact).

### Current focus

A lean set of endpoints to build a consistent, filterable time series:

- **Asking Prices** (`prices`)
- **Asking Rents (long-let)** (`rents`)
- **Sold Prices** (`sold_prices`)
- **Crime** (`crime`)

### What’s New

#### September 2025

- **Cohort-driven ETL** — Postcodes live in config/cohorts.yaml, synced into dim_cohort & dim_cohort_member.
- **Daily job** — scripts/daily_etl.py batches selected endpoints per postcode, respects vendor rate limits (4 calls / 10s), persists to SQLite, and archives JSON.
**Richer DB metadata** — Columns like request_params, run_id, context_postcode, and denormalised helpers ready for analytics modeling.
**CI workflows** — Daily ETL (scheduled + manual) and Weekly DB backup (scheduled + runs after Sunday ETL).

#### October 2025

- **Postcode-aware dedupe** — unique index on `(endpoint_name, context_postcode, checksum)`.
- **Stable checksums** — hash a projection that drops volatile fields (`process_time`, `url`) and focuses on each endpoint’s core payload.
- **Denormalised helpers** — ETL populates `anchor_postcode_*`, `effective_radius_mi`, `points_requested`, `max_age_months`, and a radius reasonableness flag.
- **Analytics views** — `v_prices`, `v_rents`, `v_sold_prices`, `v_crime` created idempotently in `init_db()`.
- **Focused endpoints** — initial dataset concentrates on: `prices`, `rents`, `sold_prices`, `crime`.
- **Cohorts** — expanded Norwich & Norfolk (NR1–NR15) seed pool.

---

## Project Structure

```bash
Property Analytics Tool/
├─ pyproject.toml                  # Packaging/build metadata (name, version, deps, console script)
├─ LICENSE                         # License file
├─ README.md                       # Project overview, setup, usage (this file)
├─ CHEAT_SHEET.md                  # Quick CLI & parameter reference (enums, bounds, credits)
├─ requirements.txt                # Optional explicit deps (requests, python-dotenv, tabulate, etc.)
├─ .env.example                    # Template for PROPERTYDATA_API_KEY (copy to .env)
├─ .gitignore                      # Ignore venvs, outputs, DBs, .env, etc.
├─ .gitattributes                  # Normalize line endings (LF/CRLF)
├─ .pre-commit-config.yaml         # ruff/black/isort hooks
├─ .github/
│  └─ workflows/
│     ├─ ci.yml                    # lint + tests on pushes/PRs
│     ├─ daily-etl.yml             # daily API fetch -> SQLite -> rolling release asset
│     └─ backup-db.yml             # weekly dated backup; also runs after Daily ETL on Sundays
│
├─ config/
│  └─ cohorts.yaml                 # authoritative postcode cohorts (e.g., “norwich-core”)
│
├─ pat/                            # Package (importable as `pat`) — core python code
│  ├─ __init__.py
│  ├─ main.py                      # CLI runner:
│  │                                # - registry(), registry_params_for_estimates()
│  │                                # - run_selected(), list_endpoints(), endpoints_for_category()
│  │                                # - Output: JSON always; optional SQLite via --save-db
│  │                                # - CLI flags: --run, --category, --save-db, --dry-run, --list-endpoints, --fallback, --town
│  │                                # - Console script entrypoint: `pat` -> `pat.main:_main`
│  │
│  ├─ constants.py                 # Single source of truth:
│  │                                # - API_ENDPOINTS mapping (friendly key → canonical API path)
│  │                                # - Enums & numeric bounds (PROPERTY_TYPE, RADIUS_MAX, etc.)
│  │                                # - DB_FILE path & defaults; POSTCODE_ONLY_ENDPOINTS
│  │
│  ├─ data_store.py                # SQLite utilities:
│  │                                # - init_db() (idempotent schema + migrations)
│  │                                # - save_response() (canonical JSON + checksum)
│  │                                # - Dedup via unique index (endpoint_name, checksum) + INSERT OR IGNORE
│  │                                # - fetch_all(), get_db_size()
│  │
│  ├─ api/
│  │  ├─ __init__.py
│  │  └─ propertydata_api.py       # API layer:
│  │                                # - loads API key from .env, retrying requests Session
│  │                                # - call_propertydata_api(), estimate_credits()
│  │                                # - get_* endpoint wrappers with validation
│  │                                # - Import-time alignment checks
│  │
│  └─ etl/
│     ├─ __init__.py
│     ├─ etl.py                    # DB-first ETL runner (process rows, track etl_state)
│     ├─ ingest_files.py           # Ingest outputs/raw/*.json → DB; archive to outputs/archived/
│     └─ retention.py              # Retention utilities: prune old files in outputs/raw & archived; optional VACUUM
│
├─ scripts/
│  ├─ __init__.py                  # marks folder as importable (for tests)
│  └─ daily_etl.py                 # Orchestrates: CLI fetch -> ingest -> ETL -> retention
│                                  # Uses PAT_RUN_ID if set for consistent run tagging
│
├─ tests/                          # Pytest suite (HTTP mocked)
│  ├─ conftest.py
│  ├─ test_api_wrappers.py         # Wrapper → core call mapping
│  ├─ test_api.py                  # API-level tests
│  ├─ test_data_store_dedup.py     # Unique-index dedup behavior
│  ├─ test_data_store.py           # DB basics
│  ├─ test_etl_runner.py           # Integration: end-to-end ETL batch run
│  ├─ test_etl.py                  # ETL helpers + run_once() smoke
│  ├─ test_ingest_files.py         # Ingest outputs/raw → DB; dedupe + archive behavior
│  ├─ test_ingest_files_edges.py   # Edge cases: bad JSON, dupes, filename/payload inference, dry-run
│  ├─ test_retention.py            # Retention pruning (raw/archived) + VACUUM hook
│  ├─ test_main_estimate_params.py # Dry-run param mirror checks
│  ├─ test_main_postcode_only.py   # Postcode-only enforcement in CLI
│  ├─ test_main.py                 # CLI behavior
│  └─ test_postcode_validation.py  # Postcode regex + postcode-only endpoint enforcement
│
├─ outputs/                        # Runtime data (git-ignored)
│  ├─ raw/                         # Timestamped JSON saved by runs
│  └─ archived/                    # Raw JSON moved here after DB save (when --save-db)
│
└─ venv/ or .venv/                 # Local virtual environment (ignored by git)
```

---

## Installation

Tested with **Python 3.13+**.

1. **Clone the repository**

   ```bash
   git clone https://github.com/ben-hardesty/property-analytics-tool.git
   cd "Property Analytics Tool"
   ```

2. **Create and activate a virtual environment** (recommended)

   ```bash
   python -m venv venv
   # On macOS / Linux: source venv/bin/activate
   # On Windows (PowerShell): venv\Scripts\Activate.ps1
   ```

3. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

4. **Create a `.env` file** in the project root with your API key:

   ```bash
   echo "PROPERTYDATA_API_KEY=your_api_key_here" > .env
   ```

---

## Quickstart (CLI)

```bash
# Run prices + rents endpoints, save JSON (and optionally SQLite)
pat --run prices rents
pat --run prices rents --save-db
```

If everything is set up correctly, you will see:

- JSON saved in `outputs/raw/`
- If `--save-db` is passed, rows inserted into `property_analytics.db`
- A run summary table printed in the terminal

---

## Cohorts & Location Management

**Why cohorts?** They let you centrally control which postcodes are fetched daily/weekly without touching code.

- Source of truth: `config/cohorts.yaml`
- Synced to DB by the ETL script:
  - `dim_cohort(cohort_id, name, description, tags, updated_at)`
  - `dim_cohort_member(cohort_id, location_type, location_key, updated_at)`

Example (Norfolk & Norwich Core):

```yaml
version: 1
cohorts:
  - id: "norfolk-norwich-core"
    name: "Norfolk & Norwich Core (NR1–NR15)"
    description: "Daily baseline cohort covering NR1–NR15: city centre (NR1–NR7) plus core Greater Norwich districts (NR8–NR15) for time-series history building."
    tags: ["daily", "baseline", "core", "norwich", "greater-norwich", "norfolk", "nr1-nr15"]
    members:
      - "NR1 1EF"
      - "NR1 3SZ"
      - "NR2 3LD"
      - "NR3 1AB"
      - "NR4 7AB"
      - "NR5 8AZ"
      - "NR6 5HT"
      - "NR7 0XE"
      - "NR8 6JR"
      - "NR9 3JJ"
      - "NR10 3DN"
      - "NR11 6EL"
      - "NR12 9AS"
      - "NR13 5LH"
      - "NR14 7WB"
      - "NR15 2XR"
```

---

## Automated ETL

### Daily ETL (`.github/workflows/daily-etl.yml`)

- Runs `scripts/daily_etl.py`
- Pulls curated endpoints daily for cohort members
- Persists to SQLite and updates a rolling GitHub Release asset (`db-latest`)
- Uses a stable run tag: `PAT_RUN_ID=${{ github.run_id }}.${{ github.run_attempt }}`

**Env overrides (optional):**

- `ETL_ENDPOINTS` — comma-separated list (default curated: sold_prices, prices, rents, crime)
- `PAT_POSTCODES_PER_DAY` — batch size per day (default 16)
- `COHORT_ID` — only use members of this cohort (e.g., `norfolk-norwich-core`)
- `POSTCODE_POOL` — explicit comma list (overrides cohorts)
- Rate limiting baked in: **≤4 calls per 10s**

### Weekly Backup (`.github/workflows/backup-db.yml`)

- Runs **Sundays 05:30 UTC** and **after Sunday’s Daily ETL**
- Makes a consistent `.backup` copy (if `sqlite3` available), zips, and uploads as an artifact (90-day retention)

---

## Running the Daily ETL locally

```bash
# Activate venv and ensure PROPERTYDATA_API_KEY is set in .env

# (Optional) override cohort or endpoints
$env:COHORT_ID="norwich-core"               # PowerShell
$env:ETL_ENDPOINTS="rents,sold_prices"
$env:PAT_POSTCODES_PER_DAY="4"

python scripts/daily_etl.py
```

The script:

1) syncs `config/cohorts.yaml` → `dim_cohort*`  
2) selects postcodes (env POSTCODE_POOL → cohort → fallback list)  
3) fetches endpoints per postcode with throttling  
4) saves JSON + SQLite, then archives JSON to `outputs/archived/`

---

## How dedupe works (important)

1) **Stable checksum**
   - Before saving, we compute a checksum on a **stable projection**:
     - Drop volatile fields (`process_time`, `url`).
     - Focus on core payload:
       - `prices` & `sold_prices`: hash `data`
       - `rents`: hash `data.long_let` (fallback `data` if needed)
       - `crime`: hash full payload minus volatile keys

2) **Postcode-aware unique key**
   - Unique index is **`(endpoint_name, context_postcode, checksum)`**.
   - Same endpoint & same postcode:
     - identical stable content ⇒ **skips insert** (`INSERT OR IGNORE`)
     - any content change ⇒ **new row**
   - Different postcodes always produce separate rows.

---

## Database Model (for analytics)

Main fact table: `api_responses`  
Key columns (subset):

- **Core:** `id`, `timestamp`, `endpoint_name`, `source`, `raw_json`, `checksum`
- **Request metadata:** `request_params`, `context_postcode`, `run_id`, `source_file`, `estimated_credits`
- **Denormalised helpers (populated by ETL):**  
  `anchor_postcode_full`, `anchor_postcode_sector`, `anchor_postcode_district`,  
  `effective_radius_mi`, `points_requested`, `max_age_months`, `bedrooms`,  
  `property_type`, `sample_from_town_center`, `too_wide_flag`
- **Processing:** `processed_at`, `processing_error`
- **Indexes:** unique `(endpoint_name, checksum)` for dedupe; lookups on time, endpoint, query_key

Cohort dims:

- `dim_cohort` and `dim_cohort_member` as described above.

---

## Built-in Analytics Views

Created idempotently by `pat.data_store.init_db()`:

- **`v_prices`**
  - Columns: `id, timestamp, postcode, average_price, points_analysed, effective_radius_mi, points_requested, too_wide_flag`
  - Source: `endpoint_name='prices'`

- **`v_rents`** (long-let)
  - Columns: `id, timestamp, postcode, average_rent_pw, points_analysed, effective_radius_mi, points_requested, too_wide_flag`
  - Source: `endpoint_name='rents'`

- **`v_sold_prices`**
  - Columns: `id, timestamp, postcode, average_sold_price, points_analysed, date_earliest, date_latest, max_age_months, effective_radius_mi, points_requested, too_wide_flag`
  - Source: `endpoint_name='sold_prices'`

- **`v_crime`**
  - Columns: `id, timestamp, postcode, population, crimes_last_12m, crimes_per_thousand, crime_rating, effective_radius_mi, points_requested`
  - Source: `endpoint_name='crime'`

> If you previously created views manually, you don’t need to delete them; `init_db()` safely **DROP VIEW IF EXISTS** then recreates the canonical versions.

---

## Current endpoint defaults (focused set)

We’ve **removed input filters** like bedrooms/property_type/max_age for now to keep the dataset broad and filter **downstream**. Each wrapper passes only the **postcode** (plus any minimal required kwargs by the API). Radius & points are chosen by the API; we persist both the **requested points** (if supplied) and the **actual points analysed**.

- `prices` — location only  
- `rents` — location only (long-let block in response)  
- `sold_prices` — location only (response includes `max_age`, `date_earliest/latest`)  
- `crime` — location only (response includes `radius` & headline stats)

---

## Usage & CLI

> **Postcode-only note:** Some endpoints require a full postcode (e.g. `politics`, `planning_applications`, `internet_speed`, etc.). The CLI enforces this.

```bash
pat [FLAGS] [ENDPOINT(S) or all]
```

Common flags:

| Flag | Description | Example |
|---|---|---|
| `--run` | Run selected endpoints. | `pat --run rents prices` |
| `--save-db` | Persist to SQLite. | `pat --run rents --save-db` |
| `--list-endpoints` | Show available endpoints. | `pat --list-endpoints` |
| `--category` | Filter by category. | `pat --category "Rental & Investment" --run all` |
| `--dry-run` | Estimate credits only. | `pat --run prices --dry-run` |

---

## Credit Safety & Rate Limiting

- Use `--dry-run` (or the daily script logs) to estimate credits before calling.
- Vendor limit enforced in ETL: **4 calls per 10 seconds**.
- Some endpoints add credits for larger result sets; see **CHEAT_SHEET.md**.

---

## ETL & Ingest (manual)

- **Ingest files → DB:** `python -m pat.etl.ingest_files`  
  Scans `outputs/raw/*.json`, infers endpoint, dedupes by `(endpoint_name, checksum)`, archives processed files.

- **DB-first ETL:** `python -m pat.etl.etl`  
  Transforms `api_responses` rows and sets `processed_at`. Extend to populate analysis-ready tables.

- **Retention:** `python -m pat.etl.retention`  
  Prunes old raw/archived files, optional `VACUUM`.

---

## Testing

```bash
pytest -q
```

- Tests are HTTP-mocked.
- Windows + sqlite resource warnings are suppressed in `tests/conftest.py`.
- New tests cover:
  - cohort YAML sync → DB dims
  - daily ETL pool selection, throttling and postcode context
  - DB schema migrations and dedupe

---

## Troubleshooting CI

- **Daily ETL “missing API key”:** Check `PROPERTYDATA_API_KEY` secret is set and non-empty.
- **Release asset clashes:** The workflow replaces (`--clobber`) the `db-latest` asset each run.
- **Weekly backup didn’t run:** It triggers Sundays 05:30 UTC and after Sunday’s Daily ETL (guarded to only proceed on Sundays).

---

## Roadmap

```bash
Infra / dev tooling
- ✅ Keep CI / pre-commit working (ruff, black, isort).
- ✅ Tests + coverage measurement integrated.
- ✅ Test fixes & new tests added and green (incl. cohorts + daily_etl script).
- ✅ GitHub Actions runs lint + tests on PRs/push (Python 3.13; pre-commit then pytest + coverage artifact).
- ✅ Status badges in README (build + coverage), coverage.xml generated by CI and badge committed.
- ✅ Stable per-run identifier wired (PAT_RUN_ID from Actions) and plumbed through to DB rows.

API wrappers & backend
- ✅ Implemented/updated `pat.api.propertydata_api` wrappers; tests mock the shared `call_propertydata_api`.
- ✅ `main.py` supports postcode default and optional town; POSTCODE_ONLY endpoints enforced.
- ✅ Helpful CLI error handling (missing API key, invalid params, timeouts) via central handler.
- ✅ `data_store.py` implemented for persistence; tests exercise DB I/O and dedupe.
- ✅ Request metadata persisted: `run_id`, `request_params`, `context_postcode`, `source_file`, `estimated_credits` (nullable).
- ✅ Denormalized helper columns created for analytics: 
     `anchor_postcode_full/sector/district`, `effective_radius_mi`, `points_requested`,
     `max_age_months`, `bedrooms`, `property_type`, `sample_from_town_center`, `too_wide_flag`.
- ✅ Populate denormalized helper columns in ETL transforms (map params & response hints to columns).

ETL, cohorts & ingestion
- ✅ Cohorts system introduced:
     - `config/cohorts.yaml` (authoritative)
     - `dim_cohort` / `dim_cohort_member` tables
     - Sync helper `pat.etl.cohorts.upsert_cohorts_from_yaml` + tests.
- ✅ Daily ETL script (`scripts/daily_etl.py`):
     - Syncs cohorts YAML → DB dims at start.
     - Selects postcodes from cohorts (env override `POSTCODE_POOL`).
     - Runs curated endpoints (env override `ETL_ENDPOINTS`).
     - Respects vendor rate limit with throttle (≤ 4 calls / 10s).
     - Persists via `pat.main` with `SAVE_TO_DB=True`.
     - Uses stable `PAT_RUN_ID` for traceability.
- ✅ Ingest (files → DB): 
     - Canonical JSON + checksum
     - Endpoint inference (filename first, then payload)
     - Dedup on (endpoint_name, checksum) with insert-or-ignore
     - Duplicate files still archived
     - Unit & edge-case tests in place.
- ✅ Retention utilities: raw/archived cleanup + optional `VACUUM` (module present; invoke from jobs when needed).

Automation & workflows
- ✅ Daily ETL workflow (`.github/workflows/daily-etl.yml`):
     - Preflight secret check (non-empty `PROPERTYDATA_API_KEY`).
     - Downloads/creates rolling DB release asset, runs ETL, re-uploads DB.
     - Sets `PAT_RUN_ID=${{ github.run_id }}.${{ github.run_attempt }}`.
- ✅ Weekly DB Backup workflow (`.github/workflows/backup-db.yml`):
     - Runs Sundays 05:30 UTC and also on `workflow_run` of Daily ETL (guarded to Sundays).
     - Produces dated zipped backup artifact (90-day retention).
- ✅ Concurrency guards to avoid overlapping runs.
- ✅ Release asset: rolling `db-latest` kept up-to-date after each Daily ETL.

Data quality & persistence
- ✅ DB migrations idempotent; schema evolved safely with additive columns.
- ✅ Unique index & dedupe tests green.
- ✅ Cohorts → dimension tables kept in sync each run.
- ⬜ Backfill strategy for historical snapshots (time-sliced pulls) within monthly credit cap.
- ⬜ Optional move of large archived JSON to object storage if repo bloat becomes a concern.

Product & analytics
- ⬜ Front-end UI (dashboards, maps, cohort switcher).
- ✅ Analytics views (asking vs sold trends, rent/sold prices, time series).
- ⬜ Cohort-aware rollups (per cohort / postcode / district) and aggregation layers.
- ⬜ Simple BI notebook examples (SQLite → pandas → charts).
- ⬜ Full historical pipeline and replays (snapshots by endpoint with cohort tags).
- ⬜ Spatial joins for district/sector aggregations and boundary overlays.

CLI / UX
- ✅ `main.py` CLI supports `--run`, `--dry-run`, `--save-db`, `--list-endpoints`, `--category`, `--fallback`.
- ✅ Dry-run credit estimates plumbed to CLI output.
- ⬜ `pat describe <endpoint>` & `pat examples <endpoint>` helpers (optional).
- ⬜ Interactive prompts for missing params (with safe defaults) when not running headless.

Platform & scale
- ⬜ Move from SQLite to Postgres or persist critical datasets to object storage.
- ⬜ Deploy backend service (containerized); server-side API key, thin client.
- ⬜ Caching/rate-limiting (per-endpoint TTLs) to reduce credits and latency. 
- ⬜ Warehouse sync (DuckDB/BigQuery) for heavier analytics.
```
