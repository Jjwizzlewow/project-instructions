# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Options Edge Seller is a full-stack application for analyzing options chains with computed metrics for wheel strategy trades (cash-secured puts and covered calls). The system fetches live options data, calculates key trading metrics, and caches results for performance.

**Tech Stack:**
- **Backend:** Python 3.10+, FastAPI, SQLite, SQLAlchemy, Pydantic, scipy (Black-Scholes), pandas_market_calendars
- **Frontend:** Next.js 15, TypeScript, React Query (TanStack Query), Tailwind CSS, shadcn/ui, Lightweight Charts v5
- **Data Sources:** Polygon.io API, yfinance, Barchart scraper
- **Ports:** Backend (localhost:8010), Frontend (localhost:3010)

## Development Setup

**Virtual Environment Location:** Project root (`.venv/`), NOT in backend folder

**Install Dependencies:**
```bash
# Backend (from project root)
pip install -r requirements.txt

# Frontend
cd frontend
pnpm install
```

**Environment Configuration:**

Create `.env` in project root:
```
POLYGON_API_KEY=your_polygon_api_key_here
```

Create `frontend/.env.local`:
```
NEXT_PUBLIC_API_BASE=http://localhost:8010
```

**Initialize Database:**
```bash
python -c "from backend.app.db.database import init_db; init_db()"
```

## Running the Application

**Backend:**
```bash
# From project root
python backend/run.py
```

**Frontend:**
```bash
cd frontend
pnpm dev
```

Access at: http://localhost:3010

## Testing

**Run All Tests:**
```bash
.venv/Scripts/python -m pytest backend/tests -q
```

**Run Specific Test File:**
```bash
.venv/Scripts/python -m pytest backend/tests/test_calc_core.py -v
```

**Run Single Test:**
```bash
.venv/Scripts/python -m pytest backend/tests/test_calc_core.py::TestCalculateDTE::test_dte_normal_case -v
```

**Test Structure:**
- `backend/tests/conftest.py` - Fixtures (TestClient, temp DB, mocked APIs)
- `backend/tests/test_api_smoke.py` - API endpoint tests
- `backend/tests/test_calc_core.py` - Calculation function tests
- `backend/tests/test_iv_service.py` - IV service unit tests
- `backend/tests/test_scraped_iv.py` - Scraped IV integration tests
- `backend/tests/test_job_queue.py` - Background job queue tests

All tests use in-process TestClient (no ports) and temporary databases. External API calls are mocked.

**Frontend Build Check:**
```bash
cd frontend
pnpm build
```

## Architecture

### Multi-Page Application Structure

The application uses Next.js App Router with URL-driven state management:
- **Home** (`/`) - Ticker input for navigation
- **Ticker** (`/ticker/[symbol]`) - Options chain analysis with price charts, IV metrics, and volatility data
- **Watchlist** (`/watchlist`) - Track multiple tickers with price + IV data
- **Portfolio** (`/portfolio`) - Stock holdings with cost basis and covered call tracking
- **Positions** (`/positions`) - Options positions management (short puts/calls)
- **Settings** (`/settings`) - Theme and cache configuration

URL parameters on ticker page: `symbol`, `expiry`, `tab`, `extra`, `itm`, `full` (all persist across refreshes)

### Backend Service Layer Architecture

Services follow single-responsibility principle with clean domain boundaries:

**Options Domain:**
- `options_service.py` - Options chain orchestration and metric calculations
- `polygon_service.py` - Polygon.io API integration (chains, snapshots, historical data)
- API-level filtering: 90-110% strike range, 25-45 DTE, 50 contract limit

**Stock Data:**
- `yfinance_service.py` - Stock prices (spot, OHLCV), earnings dates, risk-free rates
- `stock_history_service.py` - OHLCV data with Bollinger Bands calculation
- Uses unadjusted prices (`adjusted=False`) to match historical option strikes

**Volatility & IV:**
- `scraper_service.py` - Barchart scraping for live IV metrics (sequential with 2.5-4s delays)
- `volatility_scraper_service.py` - Date-based caching orchestration with 24-hour freshness
- `iv_service.py` - Historical IV computation using Black-Scholes (scipy.optimize.brentq)
  - 504-day (2-year) bootstrap for sufficient lookback data
  - Rolling 252-day windows for IV Rank and IV Percentile (stored in database)
  - First 252 days have NULL rank/percentile (insufficient history)
  - Days 253-504 have calculated values using previous 252-day windows
  - Market-aware: uses live greeks when market open, historical when closed

**Background Processing:**
- `iv_job_queue.py` - Non-blocking worker thread (threading.Thread daemon)
- Sequential job processing with database-backed queue (IvJob model)
- Lifecycle managed via FastAPI lifespan events (startup/shutdown)
- Prevents API storms when loading Watchlist/Portfolio pages

**Portfolio & Positions:**
- `portfolio_service.py` - Portfolio CRUD with duplicate detection and uppercase normalization
- `positions_service.py` - Options positions enrichment, P/L calculations, expiry reconciliation
- `market_hours_service.py` - Trading calendar using pandas_market_calendars (XNYS)

**Caching:**
- `cache_service.py` - Wraps CRUD for cache operations
- 5-minute TTL for option chains (configurable)
- 60-minute TTL for OHLCV history
- 30-minute in-process cache for same-day IV snapshots
- 24-hour date-based cache for scraped IV metrics (uses ET trading dates)

### Frontend Architecture

**State Management:**
- URL search params for UI state (ticker page)
- localStorage for settings (theme, cache TTL)
- React Query for server state (5-min to 1-hour staleTime depending on endpoint)

**Query Layer Organization:**
- `lib/query/` split by domain (options, stock, portfolio, positions, volatility, iv-queue, watchlist)
- `lib/query/hooks.ts` re-exports for convenience (import compatibility)
- Each domain file contains: TypeScript types, API functions, React Query hooks

**Component Patterns:**
- Cards-based layout on ticker page (MainPriceCard, IvCard, PriceChartCard, OptionsChainCard)
- Enriched data tables (Portfolio, Positions, Watchlist) with inline cell components
- Theme-aware styling with ThemeProvider context (light/dark/system modes)
- Lightweight Charts v5 for candlestick charts and IV historical series

### Data Flow Example (Options Chain)

1. User navigates to `/ticker/AAPL?expiry=2025-12-19`
2. `useOptionChain(ticker, expiry)` hook triggers from ticker page
3. Frontend `api.get('/api/options/chain?ticker=AAPL&expiry=2025-12-19')`
4. Backend `options.router` → `options_service.get_option_chain_with_metrics()`
5. Service checks `cache_service` (SQLite `option_chain_cache` table)
6. Cache miss: Fetch from `polygon_service` + `yfinance_service` + `scraper_service`
7. Enrich each contract with `enrich_contract()` (adds DTE, yield, expected move, % OTM)
8. Store in cache with 5-min TTL, return JSON
9. Frontend renders in `OptionsChainCard` with tabs, toggles, color-coded yields

### Database Schema

**SQLite location:** `backend/data/options.db`

**Core Tables:**
- `option_chain_cache` - Cached option chains (ticker, expiry_date, spot_price, cached_at, data_json)
- `ohlcv_cache` - Historical OHLCV data (ticker, data_json, cached_at)
- `volatility_cache` - Barchart data (ticker, IV, HV, rank, percentile, fetched_for_date, stale, source, error_log, cached_at)
- `refresh_cooldown` - 1-hour cooldown tracking for manual IV refresh

**Portfolio/Positions:**
- `portfolio` - Stock holdings (ticker, shares, cost_basis, notes, earnings_date, earnings_checked_at, added_at)
- `positions` - Options positions (symbol, option_symbol, contract_type, strike, expiry_date, qty, open_price, open_fees, status, assigned, close_price, close_fees, close_qty, closed_at, notes, open_dt)
- `watchlist` - Tracked tickers (ticker, added_at)

**IV Historical Data:**
- `iv_history` - Daily IV records (ticker, date, expiry, strike, iv, iv_rank, iv_percentile, dte, option_ticker, stock_price, option_price, risk_free_rate, created_at)
  - `iv_rank` and `iv_percentile` are NULL for first 252 days (insufficient lookback)
  - Calculated values start at day 253 using rolling 252-day windows
- `risk_free_rates` - Daily risk-free rates from yfinance ^IRX (date, rate, source, fetched_at)
- `iv_jobs` - Background job queue (id, job_type, ticker, status, progress, total_days, processed_days, error_message, queued_at, started_at, completed_at)

### Key Calculation Formulas

**Annualized Yield:**
- Puts (CSP): `(premium / strike) * (365 / dte)`
- Calls (CC): `(premium / spot) * (365 / dte)`

**Expected Move:**
- Factor: `sqrt(dte / 365)`
- Move %: `iv * factor`
- Move $: `spot * iv * factor`

**Percent OTM:**
- Puts: `(spot - strike) / spot` (negative if ITM)
- Calls: `(strike - spot) / spot` (negative if ITM)

**Premium:** Prefers `(bid + ask) / 2`, falls back to `last` price

**IV Rank:** Position of current IV in 252-day range, 0-100 scale with mid-rank ties

**IV Percentile:** Percentage of days current IV exceeded historical values, 0-100 scale with mid-rank ties

## Error Handling & Graceful Degradation

- **Polygon.io fails:** Return 500 error (don't serve stale data)
- **yfinance fails:** Log error, continue without annualized yield calculations
- **Barchart fails:** Log error, return stale data with warning flag (graceful degradation)
- **Cache write fails:** Log error, serve successfully fetched data
- **Mark price fails (positions):** Display "—" for mark and P/L

## Standalone Scripts

**Bootstrap IV History for Multiple Tickers:**
```bash
# Default ticker list (AAPL, MSFT, AMZN, NVDA, TSLA, GOOGL, META, AMD, NFLX, SPY, QQQ, IWM)
python backend/scripts/bootstrap_tickers.py --all

# Specific tickers
python backend/scripts/bootstrap_tickers.py AAPL MSFT GOOGL

# Takes ~10 minutes per ticker for 504-day bootstrap
```

## Project File Locations

**Key Reference Docs:**
- `docs/PRD.md` - Complete product specification with all formulas and schemas
- `docs/progress.md` - Current implementation snapshot
- `docs/changelog.md` - Append-only change history
- `docs/instructions/` - Architecture guides (BACKEND_INSTRUCTIONS.md, FRONTEND_INSTRUCTIONS.md, WINDOWS_DEVELOPMENT.md, etc.)

**Backend Entry:** `backend/run.py` → `backend/app/main.py` (FastAPI app with lifespan events)

**Frontend Entry:** `frontend/app/page.tsx` (home page), `frontend/app/ticker/[symbol]/page.tsx` (main workspace)

**Configuration:** `backend/app/config/settings.py` (ports, paths, cache durations)

## Development Notes

- **Virtual environment is in project root**, not in backend folder
- **requirements.txt is in project root**, not in backend folder
- Use `force_refresh=true` query param to bypass cache on options chain endpoint
- Default ticker view shows 10 closest OTM strikes; "Show Full Chain" toggle shows all
- Color-coded yields: Green (≥50%), Blue (≥25%), Gray (<25%)
- React Query handles frontend caching independently of backend cache
- Background worker thread processes IV jobs sequentially (one at a time)
- Stock splits handled correctly with unadjusted prices matching historical strikes
- Dialog components need controlled state with 100ms portal mounting delay
- Windows development: Follow `docs/instructions/WINDOWS_DEVELOPMENT.md` for path handling and process management
