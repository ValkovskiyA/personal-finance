# Personal Finance Platform

Self-built personal finance tracking and analytics platform.

## Stack
- **Warehouse**: DuckDB
- **Transformation**: dbt Core 1.8
- **Ingestion**: Python + Nordigen PSD2 API
- **Orchestration**: Prefect 3.3
- **App**: Streamlit 1.35
- **Reporting**: Power BI Desktop

## Python version
3.11.9 — use `py -3.11 -m venv .venv` to recreate the environment

## Setup
```powershell
py -3.11 -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

## Project structure
```
src/          Python source — ingestion, pipeline, app, db
data/         Seeds, exports, database file (gitignored)
dbt/          dbt models — staging, intermediate, marts
reports/      Power BI files
docs/         Architecture diagrams and notes
```