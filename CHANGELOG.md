# Changes in this update

## 1. Stale-price bug in MPT optimization (correctness)
`portfolio.py`'s `/optimize` endpoint computed each ticker's "current price" as
`max(series.values())` — the highest price in its 100-day history, not the
most recent one. Fixed to use `get_latest_price()`, matching how `alerts.py`
already did it correctly. This directly affects every weight, Sharpe ratio,
and drift alert that came out of that endpoint.

## 2. Currency mixing (correctness)
The app never distinguished USD tickers (AAPL) from INR tickers
(RELIANCE.BSE) — totals, the allocation pie chart, and the MPT optimizer all
summed/compared raw numbers regardless of currency.

Added `app/core/currency.py`:
- `get_currency(ticker)` — infers currency from the `.BSE`/`.NSE` suffix.
- `dominant_currency_group(holdings)` — for features that can only reason
  about one currency at a time (optimize, alerts), picks the currency with
  the most holdings and reports what got excluded.

Changes:
- `/portfolio/value` now returns `totals` as a list broken out **per
  currency**, instead of one flat (and previously incorrect) total.
- `/portfolio/optimize` and `/alerts/` now run only on the dominant currency
  group and note excluded tickers in `warning`.
- `ai.py`'s portfolio context uses the correct symbol per ticker instead of
  hardcoding ₹, and flags the combined totals as approximate when currencies
  are mixed.
- `Dashboard.jsx` shows one stat-box group per currency, formats each row
  with its own currency, and scopes the allocation pie to the dominant
  currency.

**Not done, by design:** live FX conversion. Doing this properly means a
real-time exchange-rate lookup and converting before aggregating — a bigger
feature than a "fix." What's here now is honest segmentation (never silently
mixing currencies) rather than a false single number. `/portfolio/history`
still sums raw values across currencies for now — flagged in its docstring
as a known simplification, since building per-currency series felt like
scope creep here; worth doing properly as a fast follow if you'll actually
run mixed portfolios day to day.

## 3. Security
- `security.py`: `JWT_SECRET_KEY` no longer falls back to
  `"change-this-in-production"`. The backend now raises at startup if it's
  unset, so an insecure default can't silently ship. `docker-compose.yml`
  and `.env.example` updated to match — no default provided there either.
- `main.py`: CORS origins are now read from `CORS_ORIGINS` (comma-separated),
  defaulting to `http://localhost:3000` instead of `["*"]`. Tighten this to
  your real domain before deploying anywhere public.
- `price_sync.py`: manual `/portfolio/sync-prices` calls are now rate-limited
  to once per 5 minutes (`MIN_SECONDS_BETWEEN_SYNCS`), so a few friends
  clicking "Refresh prices" close together won't burn through the shared
  25-requests/day Alpha Vantage quota. The scheduled daily job bypasses this
  with `force=True`.
- Still open (not code-fixable, action items for you): rotate the Alpha
  Vantage and Gemini keys that were pasted into this chat, and confirm
  `.env` is gitignored before this touches a repo.

## 4. Smaller correctness fixes
- `models.py`: `datetime.utcnow()` (deprecated) replaced with
  `datetime.now(timezone.utc)` for `created_at`/`fetched_at` defaults.
- `holdings.py`: re-uploading the same CSV, or adding to an existing
  position, now merges into the existing holding using a shares-weighted
  average cost basis instead of creating a duplicate row. **Trade-off:**
  this collapses multiple purchase lots into one averaged position — you
  lose per-lot cost-basis history. If you need real tax-lot tracking, this
  isn't the right model; you'd want a separate `lots` table instead. Flagged
  in the code as a deliberate design choice, not an oversight.
- `alerts.py` / `portfolio.py`: both already fell back to `cost_basis` as a
  stand-in market value before prices are synced. Left as-is functionally,
  but now scoped per-currency (see #2) so that fallback doesn't compound
  with the currency-mixing bug.

## 5. Engineering hygiene
- `alpha_vantage.py`, `price_sync.py`, `scheduler.py`: replaced bare
  `print()` calls with the standard `logging` module; `main.py` calls
  `logging.basicConfig()` once at startup.
- `requirements.txt`: added `pytest`.
- `backend/tests/test_csv_parser.py`, `backend/tests/test_mpt.py`: a
  starter suite covering the ticker-regex fix, CSV validation edge cases,
  and MPT's minimum-history guard. Run with `pytest` from `backend/`.
  **Not comprehensive** — no tests for the routers/DB layer yet (would need
  a test DB fixture); this covers the two modules with pure-function logic
  that already had real bugs.

**Not done:** Alembic migrations. You're still using
`Base.metadata.create_all()`, which only creates missing tables — it won't
alter existing ones. This is fine today but will bite the first time you
change a column on a table that already has data in it. Setting up Alembic
properly (env.py, initial revision matching current schema, etc.) is real
setup work I didn't want to do silently as a drive-by change; worth doing
deliberately when you're ready, ideally before your friends' data has grown
large enough that a manual `ALTER TABLE` feels risky.

## Files touched
```
backend/app/core/currency.py        (new)
backend/app/core/security.py
backend/app/core/price_sync.py
backend/app/core/scheduler.py
backend/app/core/alpha_vantage.py
backend/app/main.py
backend/app/models/models.py
backend/app/routers/holdings.py
backend/app/routers/portfolio.py
backend/app/routers/alerts.py
backend/app/routers/ai.py
backend/requirements.txt
backend/tests/test_csv_parser.py    (new)
backend/tests/test_mpt.py           (new)
frontend/src/Dashboard.jsx
docker-compose.yml
.env.example
```

## To apply
1. Copy these files into your project at the matching paths, overwriting
   the originals.
2. Make sure your `.env` has a real `JWT_SECRET_KEY` (`openssl rand -hex
   32`) — the backend will refuse to start otherwise.
3. `docker compose up --build` (rebuild needed since `requirements.txt`
   changed).
4. Optional: `docker compose exec backend pytest` to run the new test suite.
