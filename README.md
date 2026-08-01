# S&P 500 RSI Dashboard

A Streamlit app that charts RSI(14) for S&P 500 stocks, with per-user login, a
virtual portfolio (buy/track holdings against live prices), and an AI report
generator (Dify workflow) per symbol. See `User Manual - Automated Stock
Market Application.docx` for the click-through walkthrough.

## Architecture

- `app.py` — Streamlit entry point: auth, symbol picker, page layout, dialogs
  (chatbot, virtual purchase).
- `data.py` — all persistence. **PostgreSQL only** — users, accounts,
  holdings, and a price cache all live in Postgres (via SQLAlchemy). Price
  data comes from `yfinance`, cached to Postgres, with Postgres itself as the
  fallback source if a live fetch fails.
- `indicators.py` — RSI(14) calculation (Wilder's smoothing).
- `dashboard.py` — Plotly chart building + the per-symbol UI row.
- `api.py` — Dify workflow HTTP client (streaming) for the AI report feature.
- `telemetry.py` — optional Azure Application Insights logging; no-ops if
  `APPLICATIONINSIGHTS_CONNECTION_STRING` isn't set.

Postgres is required — there's no SQLite fallback for the app tables (an
earlier `local_trading_data.db` in this folder is a leftover from before that
refactor and isn't used by the current code).

## Required configuration

Two values, from environment variables **or** `.streamlit/secrets.toml`:

```
DATABASE_URL=postgresql://user:password@host:5432/dbname
DIFY_API_KEY=app-...   # optional — only needed for the "Open chatbot" report feature
```

Copy `.env.example` to `.env` for local runs, or fill in
`.streamlit/secrets.toml` (gitignored either way — never commit real values).

## Run locally

```powershell
.\venv\Scripts\python -m pip install -r requirements.txt
.\venv\Scripts\python -m pytest -q          # unit tests mock Postgres, no DB needed
.\venv\Scripts\python -m streamlit run app.py
```

You need a reachable Postgres instance for the app itself to run (tests don't
need one — see `TEST_PROCEDURE.md`). On first run it creates its own tables
and seeds a `demo_user` / `demo` login.

## Deploying

**App host:** [Streamlit Community Cloud](https://share.streamlit.io) — free,
deploys straight from a GitHub repo, and its Secrets manager writes
`.streamlit/secrets.toml` for you (set `DATABASE_URL` and `DIFY_API_KEY`
there).

**Database:** any hosted Postgres works (free tiers: Neon, Supabase, Render
Postgres). Use the connection string it gives you as `DATABASE_URL`.

A `Dockerfile` is also included if you'd rather run this as a container
(Render, Fly.io, etc.) — it just needs `DATABASE_URL` set as an env var on
whatever host you pick.

## Known fixes applied (previously broken)

- `app.py` had `_resolve_database_url()` hardcoded to
  `postgresql://postgres:postgres@localhost:5432/postgres`, ignoring
  `DATABASE_URL`/secrets entirely — restored to read from env/secrets.
- Secrets folder was misspelled `.steamlit/` (Streamlit only reads
  `.streamlit/`) — renamed.
- `requests` (used directly by `api.py`) was missing from `requirements.txt`
  — added explicitly.
