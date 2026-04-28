# Phase Decisions — Personal Finance Platform

## 1. Architecture decision

**Decision:** Build a custom solution (Scenario B — Modern Engineer Stack)
**Rejected:** GnuCash + custom enhancements (Scenario A)

**Reasons for rejection of GnuCash:**
- Envelope / virtual account concept does not map cleanly to GnuCash's paradigm
- Budget versioning is not natively supported
- No web / mobile access — wife's dashboard requirement needs a separate layer anyway
- Spanish bank PSD2 integration adds complexity GnuCash does not help with
- Extending GnuCash requires poorly documented GObject introspection Python bindings
- Would spend time fighting GnuCash's architecture instead of building requirements

**Reasons for custom build:**
- Full control over data model, transformations, and UI
- Every layer of the stack is a transferable skill (directly applicable at Novartis)
- Modern data engineering stack mirrors enterprise production patterns
- All tools are free and open source
- Dual objective satisfied: automate personal finance AND learn modern stack

---

## 2. Technology stack decisions

### Database — DuckDB over PostgreSQL or SQLite

**Decision:** DuckDB 1.1.3
**Rejected:** PostgreSQL, SQLite

**Reasons:**
- No server to install or manage — single `.duckdb` file on disk
- Full SQL including window functions and recursive CTEs for account hierarchy
- Columnar storage — fast analytical queries without indexes
- Native CSV and Parquet reading — useful for seed loads
- dbt-duckdb adapter is mature and well maintained
- Can be queried directly from Python, dbt, Power BI, and Streamlit

### Transformation — dbt Core over pure Python

**Decision:** dbt-core 1.8.0 + dbt-duckdb 1.8.1
**Rejected:** Pure Python pandas transformations

**Reasons:**
- Version-controlled SQL models with full lineage
- Built-in testing framework for data quality
- Staging / intermediate / mart separation enforces clean architecture
- Industry standard — directly transferable to enterprise data engineering
- Seeds handle versioned budget CSV loads natively
- Jinja templating enables reusable macros

### Orchestration — Prefect over Airflow or cron

**Decision:** Prefect 3.3.0
**Rejected:** Prefect 2.x (compatibility issues), Airflow (too heavy for personal use)

**Reasons:**
- Prefect 3.x has clean Python 3.11 compatibility
- Simpler API than Airflow, lighter weight
- Local process runner — no server required for basic scheduling
- Good observability dashboard for monitoring daily pipeline runs
- Windows Task Scheduler retained as fallback if Prefect causes issues

**Note:** Prefect 2.19.x was attempted and rejected due to pydantic compatibility
conflicts on Python 3.11. Prefect 3.1.0 also had issues. 3.3.0 is the confirmed
working version.

### Application layer — Streamlit over Flask or FastAPI

**Decision:** Streamlit 1.35.0
**Rejected:** Flask, FastAPI, Django

**Reasons:**
- Python-native — no HTML / CSS / JavaScript required
- Hot reload during development — edit `.py` file, browser updates instantly
- Sufficient for transaction review UI, envelope management, budget overview
- Directly queryable against DuckDB mart models
- Fast to build — focus stays on data logic not web framework boilerplate

### Reporting — Power BI Desktop + Power BI Service

**Decision:** Power BI Desktop for personal analytics, Power BI Service for wife's dashboards
**Alternatives considered:** Evidence.dev (code-first, kept as secondary option)

**Reasons:**
- Power BI Desktop is free and connects to DuckDB via ODBC driver
- Power BI Service free tier sufficient for sharing read-only dashboards
- DAX enables account hierarchy drill-down directly from mart_account_hierarchy
- Existing skill — no new learning curve for the reporting layer
- Wife gets mobile-friendly read-only access without installing any software

### Bank integration — Nordigen / GoCardless API

**Decision:** Nordigen (GoCardless) PSD2 API
**Reasons:**
- Free tier available
- Covers Spanish banks (BBVA, Santander, CaixaBank supported)
- Standard PSD2 Open Banking API — well documented
- Returns structured JSON transaction data

### Notifications — Gmail API

**Decision:** Gmail API via google-api-python-client
**Purpose:** Send unclassified transaction assignment requests
**Trigger:** Prefect detects unclassified rows in int_classified_txns on daily run

---

## 3. Python version decision

**Decision:** Python 3.11.9
**Rejected:** Python 3.13.1 (originally installed)

**Reasons:**
- Python 3.13 released October 2024 — too new for the data engineering ecosystem
- numpy 1.26.x has no wheels for Python 3.13 — requires C++ build tools to compile
- pandas 2.2.x same issue on 3.13
- Prefect 2.x and 3.x both had pydantic compatibility issues on 3.13
- Python 3.11 is the sweet spot — full wheel availability for entire stack
- dbt, Prefect, Streamlit all have extensive test coverage against 3.11

**Important:** Virtual environment must be created with:
```powershell
py -3.11 -m venv .venv
```
Not plain `python -m venv .venv` — this ensures 3.11 is used even if 3.13
remains the system default.

---

## 4. Data model decisions

### Double-entry bookkeeping as the transactional core

**Decision:** journal_entry + journal_line pattern (classic double-entry ledger)
**Rejected:** Single-entry transaction table (one row per transaction)

**Reasons:**
- Double-entry is the accounting standard — books always balance
- Supports credit card liability tracking correctly (expense posted at purchase,
  liability settled at payment — two separate economic events)
- Append-only design preserves complete audit trail
- Corrections posted as reversing entries — no data mutation
- Compatible with standard accounting reports (P&L, balance sheet)

### Envelopes as analytical overlay on journal lines

**Decision:** envelope_id as optional foreign key on journal_line
**Rejected:** Separate envelope ledger with its own transaction table

**Reasons:**
- Keeps double-entry books clean — no phantom balances
- Envelope balance always derived from journal lines — single source of truth
- No risk of envelope ledger drifting out of sync with main ledger
- Envelope is an analytical concept, not an accounting one — belongs in the
  analytical layer, not the transactional core

### Account hierarchy via adjacency list (self-referencing table)

**Decision:** parent_account_id FK references account_id in the same table
**Reasons:**
- Simple to load and maintain via CSV seed
- DuckDB recursive CTEs handle traversal and rollup efficiently
- Supports unlimited depth (project uses up to 5 levels)
- Standard pattern — well understood, easy to query

### Budget versioning via budget_version table

**Decision:** Separate budget_version table with is_current flag
**Reasons:**
- Every CSV load is preserved permanently — full history of budget changes
- stg_budget_lines filters to is_current = true by default
- Historical versions always queryable for variance analysis
- Budget can be updated and reloaded at any time without losing prior versions

---

## 5. Project structure decisions

### Single Git repository for entire project

**Decision:** All code, dbt models, seeds, Streamlit app, and pipeline in one repo
**Reasons:**
- Personal project — monorepo is simpler to manage
- dbt models, Python scripts, and config files are tightly coupled
- Single commit can span a schema change + dbt model update + app change

### Secrets management via .env file

**Decision:** python-dotenv + .env file, gitignored
**Reasons:**
- Simple, no external secret manager needed for personal project
- .env.example committed to Git as documentation of required variables
- Nordigen API keys and Gmail credentials never touch version control
- DUCKDB_PATH kept in .env so path can differ between environments

### Database file gitignored

**Decision:** *.duckdb added to .gitignore
**Reasons:**
- Contains real personal financial data — privacy
- Binary file — Git cannot diff it meaningfully
- Full copy stored on every commit — repository bloat
- Reproducible from source: init_db.py + dbt run recreates it completely

---

## 6. Development environment decisions

### VS Code as primary IDE

**Decision:** VS Code with extensions
**Key extensions installed:**
- Python (Microsoft)
- Pylance (Microsoft)
- dbt Power User (Altimate AI) — NOT the official dbt Labs extension
- SQLTools + DuckDB Driver
- Jupyter
- GitLens
- YAML (Red Hat)
- Rainbow CSV
- Even Better TOML

**Note:** dbt Labs official extension was evaluated and rejected in favour of
dbt Power User by Altimate AI — significantly more features including lineage
graph, inline query preview, and one-click run/test.

### One chat per phase workflow

**Decision:** Start a fresh Claude chat for each phase with a structured handoff message
**Reasons:**
- Keeps context focused on current phase only
- Phase 1 debugging sessions (package conflicts, Python version) are noise for Phase 2+
- Each chat becomes a self-contained reference document for that phase
- Reduces token consumption per message

**Handoff template location:** See README.md or beginning of each phase chat

---

## 7. Confirmed working package versions
Python        3.11.9
duckdb        1.1.3
dbt-core      1.8.0
dbt-duckdb    1.8.1
requests      2.32.3
numpy         1.26.4
pandas        2.2.3
openpyxl      3.1.2
prefect       3.3.0
streamlit     1.35.0
plotly        5.22.0
google-auth               2.29.0
google-auth-oauthlib      1.2.0
google-api-python-client  2.130.0
python-dotenv  1.0.1
pyyaml         6.0.1
loguru         0.7.2


---

## 8. Decisions deferred to later phases

| Decision | Deferred to |
|---|---|
| Nordigen API registration and credentials | Phase 5 |
| Gmail API OAuth setup | Phase 8 |
| Power BI ODBC driver installation | Phase 9 |
| Windows Task Scheduler as Prefect fallback | Phase 8 |
| Evidence.dev setup as secondary reporting layer | Phase 9 (optional) |