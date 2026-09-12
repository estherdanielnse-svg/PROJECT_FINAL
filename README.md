Link:  https://projectfinal-kgxym3xauuaptmxysxzmns.streamlit.app/
 capstone project satisfying all four required components:

| # | Requirement | Implementation |
|---|---|---|
| 1 | Automated Data Ingestion | `ingestion.py` — polls the free, key-less **Open-Meteo Air Quality API** on a schedule and writes to DuckDB |
| 2 | In-Memory Analytical Queries | DuckDB SQL (`GROUP BY date_trunc('hour', ...)`) in `app.py` cleans + summarizes raw readings in real time |
| 3 | Visual Uncertainty Forecasts | `compute_forecast()` — OLS linear trend + a statistically derived prediction interval, plotted as a shaded band in Plotly |
| 4 | Live Public Deployment | Single Streamlit app deployable to Streamlit Community Cloud (or any host) — see below |

## Architecture

```
Open-Meteo Air Quality API
        │  (requests, every 15 min)
        ▼
ingestion.py  ──►  air_quality.duckdb   (DuckDB file, single source of truth)
   ▲ runs as a daemon thread                  │
   │ started automatically by app.py          │  SQL (GROUP BY, avg, date_trunc)
   │                                           ▼
   └──────────────────────────────────  app.py (Streamlit)
                                           │
                                           ├─ KPI metrics (current / 24h / 7d)
                                           ├─ Plotly historical + forecast chart
                                           │   with shaded 95% confidence band
                                           └─ Raw hourly summary table
```

**Why one process does both jobs:** DuckDB only supports a single writer
against an on-disk file at a time. Running ingestion as a background
`threading.Thread` *inside* the Streamlit process (see
`start_background_ingestion` in `app.py`, gated behind `st.cache_resource`
so it starts exactly once) keeps everything in one deployable app — no
second worker process or hosted job scheduler is required for the demo.
`ingestion.py` also runs standalone (`python ingestion.py`) if you'd rather
run ingestion as a separate always-on process/cron job in production.

## Local setup

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
streamlit run app.py
```

On first load the app automatically:
1. Creates `air_quality.duckdb` and its schema.
2. Backfills ~3 days of hourly history so the charts aren't empty.
3. Starts polling live readings every 15 minutes in the background.

Use the sidebar to switch location/pollutant, change the forecast horizon
and confidence level, or force an immediate refresh.

## Deploying publicly (Streamlit Community Cloud)

1. Push this folder to a public (or private) GitHub repo.
2. Go to **share.streamlit.io** → "New app" → point it at the repo,
   branch, and `app.py`.
3. Deploy. Streamlit Cloud installs `requirements.txt` automatically and
   gives you a public `https://<app-name>.streamlit.app` URL.
4. `air_quality.duckdb` is created inside the container's ephemeral
   filesystem — it persists for the life of that container instance and
   rebuilds itself (backfill + live polling) on cold restarts, so the
   dashboard is always populated within seconds of a redeploy.

**Alternative hosts:** Render, Railway, Fly.io, or Hugging Face Spaces all
run a plain `streamlit run app.py` the same way — just set the start
command accordingly.

**Scaling beyond the demo:** for a production deployment with guaranteed
durability, swap the in-process background thread for a real scheduled job
(cron, GitHub Actions, or a hosted worker) that runs `python ingestion.py`
independently and writes to a persistent volume or a hosted DuckDB/MotherDuck
database, and point `app.py` at that same path/connection string.

## Files

- `ingestion.py` — API client, DuckDB schema, historical backfill, polling loop
- `app.py` — Streamlit dashboard, SQL analytics, forecast model, Plotly charts
- `requirements.txt` — pinned dependencies
- `.streamlit/config.toml` — custom dark theme

