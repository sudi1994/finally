# FinAlly — Implementation Plan (Claude's Plan)

## Status Overview

The market data subsystem (`backend/app/market/`) is **complete and tested** (73 tests, 84% coverage).
Everything else — database, portfolio API, watchlist API, chat/LLM integration, frontend, and Docker — remains to be built.

This document defines the implementation order, component design, and conventions to follow.

---

## 1. Implementation Order

Build in this sequence. Each phase produces a working, testable artifact before the next begins.

```
Phase 1 → Database layer + FastAPI app shell
Phase 2 → Portfolio & Watchlist API
Phase 3 → LLM Chat API
Phase 4 → Frontend
Phase 5 → Docker & scripts
Phase 6 → E2E tests
```

---

## 2. Phase 1 — Database Layer + App Shell

### 2.1 Backend dependencies to add (pyproject.toml)

```
aiosqlite>=0.20.0       # async SQLite driver
litellm>=1.50.0         # LLM routing (→ OpenRouter → Cerebras)
python-dotenv>=1.0.0    # .env loading
httpx>=0.27.0           # async HTTP (used by LiteLLM and tests)
```

### 2.2 Directory layout for new backend code

```
backend/app/
├── __init__.py
├── main.py               # FastAPI app factory, lifespan, static file serving
├── config.py             # Settings from env vars (pydantic-settings or os.getenv)
├── database.py           # DB connection pool, lazy init, schema creation
├── market/               # (already complete)
├── api/
│   ├── __init__.py
│   ├── portfolio.py      # /api/portfolio routes
│   ├── watchlist.py      # /api/watchlist routes
│   └── chat.py           # /api/chat route
└── services/
    ├── __init__.py
    ├── portfolio.py      # Trade execution, P&L, snapshot logic
    ├── watchlist.py      # Watchlist CRUD
    └── chat.py           # LLM call, structured output parsing, action execution
```

### 2.3 Database schema (backend/db/schema.sql)

```sql
CREATE TABLE IF NOT EXISTS users_profile (
    id TEXT PRIMARY KEY DEFAULT 'default',
    cash_balance REAL NOT NULL DEFAULT 10000.0,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS watchlist (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    ticker TEXT NOT NULL,
    added_at TEXT NOT NULL,
    UNIQUE (user_id, ticker)
);

CREATE TABLE IF NOT EXISTS positions (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    ticker TEXT NOT NULL,
    quantity REAL NOT NULL,
    avg_cost REAL NOT NULL,
    updated_at TEXT NOT NULL,
    UNIQUE (user_id, ticker)
);

CREATE TABLE IF NOT EXISTS trades (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    ticker TEXT NOT NULL,
    side TEXT NOT NULL CHECK (side IN ('buy', 'sell')),
    quantity REAL NOT NULL,
    price REAL NOT NULL,
    executed_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS portfolio_snapshots (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    total_value REAL NOT NULL,
    recorded_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS chat_messages (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    role TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    actions TEXT,
    created_at TEXT NOT NULL
);
```

Seed data: one `users_profile` row and ten `watchlist` rows (AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX).

### 2.4 database.py design

- Uses `aiosqlite` with a single async connection shared via app state
- `init_db(db_path)` — creates tables from schema, seeds default data if users_profile is empty
- `get_db()` — FastAPI dependency returning the connection
- DB path defaults to `/app/db/finally.db`; configurable via `DB_PATH` env var

### 2.5 main.py design

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 1. Init database
    # 2. Create PriceCache
    # 3. Load watchlist tickers from DB, start market data source
    # 4. Start portfolio snapshot background task (every 30s)
    # 5. Store everything in app.state
    yield
    # Shutdown: stop market data source, close DB

app = FastAPI(lifespan=lifespan)
app.include_router(stream_router, prefix="/api")
app.include_router(portfolio_router, prefix="/api")
app.include_router(watchlist_router, prefix="/api")
app.include_router(chat_router, prefix="/api")
app.mount("/", StaticFiles(directory="static", html=True), name="static")
```

---

## 3. Phase 2 — Portfolio & Watchlist API

### 3.1 Portfolio endpoints

**GET /api/portfolio**
```json
{
  "cash_balance": 8500.00,
  "total_value": 11200.00,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 5,
      "avg_cost": 190.00,
      "current_price": 195.00,
      "market_value": 975.00,
      "unrealized_pnl": 25.00,
      "unrealized_pnl_pct": 2.63
    }
  ]
}
```

**POST /api/portfolio/trade**

Request: `{"ticker": "AAPL", "quantity": 5, "side": "buy"}`

Validation:
- `buy`: `quantity * current_price <= cash_balance`; deduct cash, upsert position with updated avg_cost
- `sell`: position must exist with `quantity >= requested`; add proceeds to cash, reduce/remove position
- Both: insert trade record, take immediate portfolio snapshot

Response: updated portfolio state or `{"error": "..."}` with HTTP 400.

**GET /api/portfolio/history**

Returns array of `{total_value, recorded_at}` for the P&L chart. Limits to last 500 snapshots.

### 3.2 Average cost calculation for buys

```
new_avg_cost = (existing_qty * existing_avg_cost + new_qty * fill_price)
               / (existing_qty + new_qty)
```

### 3.3 Watchlist endpoints

**GET /api/watchlist** — returns tickers with current price, change, direction from PriceCache.

**POST /api/watchlist** — adds ticker, calls `source.add_ticker(ticker)` to begin price updates, returns 409 if duplicate.

**DELETE /api/watchlist/{ticker}** — removes from DB and calls `source.remove_ticker(ticker)`.

### 3.4 Portfolio snapshot background task

```python
async def snapshot_task(app_state):
    while True:
        await asyncio.sleep(30)
        await record_snapshot(app_state.db, app_state.cache)
```

`record_snapshot` reads all positions, prices from cache, computes total = cash + sum(qty * price), inserts into `portfolio_snapshots`.

---

## 4. Phase 3 — LLM Chat API

### 4.1 POST /api/chat

Request: `{"message": "Buy 5 shares of AAPL"}`

Response:
```json
{
  "message": "Done! Bought 5 shares of AAPL at $195.00. Your cash balance is now $8,525.",
  "trades": [{"ticker": "AAPL", "side": "buy", "quantity": 5}],
  "watchlist_changes": [],
  "trade_results": [{"ticker": "AAPL", "side": "buy", "quantity": 5, "success": true, "price": 195.00}]
}
```

### 4.2 LLM call (services/chat.py)

Uses the cerebras-inference skill pattern: LiteLLM → OpenRouter → `openrouter/openai/gpt-oss-120b` with Cerebras inference.

```python
import litellm

response = await litellm.acompletion(
    model="openrouter/openai/gpt-oss-120b",
    api_key=settings.OPENROUTER_API_KEY,
    api_base="https://openrouter.ai/api/v1",
    messages=messages,
    response_format={"type": "json_object"},
    extra_body={"provider": {"order": ["Cerebras"]}}
)
```

Parse `response.choices[0].message.content` as JSON matching the structured output schema.

### 4.3 Context assembly

Before each LLM call, build a system message containing:
- Cash balance
- All positions with current price and unrealized P&L
- Watchlist with current prices
- Total portfolio value
- Last 20 chat messages from DB for conversation continuity

### 4.4 Structured output schema

```python
from pydantic import BaseModel

class TradeAction(BaseModel):
    ticker: str
    side: str  # "buy" | "sell"
    quantity: float

class WatchlistChange(BaseModel):
    ticker: str
    action: str  # "add" | "remove"

class LLMResponse(BaseModel):
    message: str
    trades: list[TradeAction] = []
    watchlist_changes: list[WatchlistChange] = []
```

### 4.5 Mock mode

When `LLM_MOCK=true`, return a hardcoded deterministic response without calling OpenRouter:
```json
{
  "message": "Mock response: I can help you manage your portfolio.",
  "trades": [],
  "watchlist_changes": []
}
```

### 4.6 Action execution order

1. Execute watchlist additions (so subsequent trades can reference new tickers)
2. Execute each trade in order (same validation as manual trades)
3. Execute watchlist removals
4. Collect results/errors for each action
5. Store assistant message + actions JSON in `chat_messages`

---

## 5. Phase 4 — Frontend

### 5.1 Next.js setup

```
frontend/
├── package.json          # next, react, typescript, tailwindcss, lightweight-charts, recharts
├── next.config.ts        # output: 'export', basePath: ''
├── tailwind.config.ts    # custom dark theme colors
├── tsconfig.json
└── src/
    ├── app/
    │   ├── layout.tsx    # root layout, global CSS
    │   └── page.tsx      # single page — imports all panels
    ├── components/
    │   ├── Header.tsx
    │   ├── Watchlist.tsx
    │   ├── MainChart.tsx
    │   ├── PortfolioHeatmap.tsx
    │   ├── PnLChart.tsx
    │   ├── PositionsTable.tsx
    │   ├── TradeBar.tsx
    │   └── ChatPanel.tsx
    ├── hooks/
    │   ├── usePriceStream.ts   # EventSource → price state
    │   ├── usePortfolio.ts     # polling /api/portfolio
    │   └── useChat.ts          # chat state + POST /api/chat
    └── lib/
        ├── api.ts              # typed fetch wrappers
        └── types.ts            # shared TypeScript interfaces
```

### 5.2 Layout (CSS Grid)

```
┌─────────────── Header (total value, cash, connection dot) ───────────────┐
│ Watchlist (left │ Main Chart (center, tall)        │ Chat Panel (right)  │
│ ~20% width)     │                                  │ ~25% width)         │
│                 ├──────────────────────────────────┤                     │
│                 │ Portfolio Heatmap │ P&L Chart     │                     │
│                 │ (bottom left)    │ (bottom right) │                     │
└─────────────────┴──────────────────┴────────────────┴─────────────────────┘
│                         Positions Table + Trade Bar                       │
└───────────────────────────────────────────────────────────────────────────┘
```

### 5.3 SSE hook (usePriceStream.ts)

```typescript
const es = new EventSource('/api/stream/prices')
es.onmessage = (event) => {
  const update: PriceUpdate = JSON.parse(event.data)
  setPrices(prev => ({ ...prev, [update.ticker]: update }))
  // Append to sparkline history (cap at 100 points per ticker)
  setSparklines(prev => ({
    ...prev,
    [update.ticker]: [...(prev[update.ticker] ?? []), update.price].slice(-100)
  }))
  // Trigger flash animation
  setFlashing(prev => ({ ...prev, [update.ticker]: update.direction }))
  setTimeout(() => setFlashing(prev => { const n = {...prev}; delete n[update.ticker]; return n }), 500)
}
```

### 5.4 Price flash CSS

```css
.flash-up   { animation: flash-green 500ms ease-out }
.flash-down { animation: flash-red   500ms ease-out }

@keyframes flash-green { 0% { background: #16a34a } 100% { background: transparent } }
@keyframes flash-red   { 0% { background: #dc2626 } 100% { background: transparent } }
```

### 5.5 Charting

- **Sparklines** in the watchlist: inline `<canvas>` drawn manually (no library needed for simple lines)
- **Main chart**: TradingView Lightweight Charts (`lightweight-charts`) — line series, auto-sized via ResizeObserver
- **Portfolio heatmap**: `recharts` Treemap component — each rect colored from red (#dc2626) through neutral to green (#16a34a) by P&L %
- **P&L chart**: `recharts` AreaChart fed from `/api/portfolio/history`

### 5.6 Tailwind color config

```typescript
// tailwind.config.ts
colors: {
  bg: { DEFAULT: '#0d1117', panel: '#1a1a2e' },
  border: '#2a2a3e',
  accent: { yellow: '#ecad0a', blue: '#209dd7', purple: '#753991' },
  up: '#16a34a',
  down: '#dc2626',
}
```

---

## 6. Phase 5 — Docker & Scripts

### 6.1 Dockerfile (multi-stage)

```dockerfile
# Stage 1: build Next.js static export
FROM node:20-slim AS frontend-builder
WORKDIR /frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# Stage 2: Python runtime
FROM python:3.12-slim
WORKDIR /app
RUN pip install uv
COPY backend/pyproject.toml backend/uv.lock ./
RUN uv sync --frozen --no-dev
COPY backend/ ./
COPY --from=frontend-builder /frontend/out ./static
RUN mkdir -p /app/db
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 6.2 docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - finally-data:/app/db
    env_file: .env

volumes:
  finally-data:
```

### 6.3 start_mac.sh

```bash
#!/usr/bin/env bash
set -e
IMAGE=finally
if [[ "$1" == "--build" ]] || ! docker image inspect $IMAGE &>/dev/null; then
  docker build -t $IMAGE .
fi
docker run -d --name finally -v finally-data:/app/db -p 8000:8000 --env-file .env $IMAGE
echo "FinAlly running at http://localhost:8000"
open http://localhost:8000 2>/dev/null || true
```

### 6.4 .env.example

```
OPENROUTER_API_KEY=your-openrouter-api-key-here
MASSIVE_API_KEY=
LLM_MOCK=false
```

---

## 7. Phase 6 — Testing

### 7.1 Backend unit tests to add

New test modules alongside `tests/market/` (already complete):

| Module | What to test |
|--------|-------------|
| `tests/test_database.py` | Schema creation, seed data, idempotent init |
| `tests/test_portfolio.py` | Trade execution, avg_cost math, P&L calc, validation errors |
| `tests/test_watchlist.py` | CRUD, duplicate handling |
| `tests/test_chat.py` | Structured output parsing, mock mode, action execution |
| `tests/test_api.py` | Route status codes, response shapes via httpx TestClient |

### 7.2 Frontend unit tests

Using `@testing-library/react` + `vitest`:

- `Watchlist.test.tsx` — renders tickers, flash class applied on price change
- `TradeBar.test.tsx` — buy/sell form submission
- `ChatPanel.test.tsx` — message send, loading state, response rendering

### 7.3 E2E tests (test/)

```
test/
├── docker-compose.test.yml   # app + playwright containers
├── playwright.config.ts
└── tests/
    ├── watchlist.spec.ts     # add/remove tickers
    ├── trading.spec.ts       # buy → cash decreases, sell → position clears
    ├── portfolio.spec.ts     # heatmap renders, P&L chart has data
    └── chat.spec.ts          # send message, mock response, inline trade confirmation
```

`docker-compose.test.yml` sets `LLM_MOCK=true` so tests run without an API key.

---

## 8. Key Implementation Notes

### Error handling
- All trade validation errors return HTTP 400 with `{"error": "..."}` — never 500
- LLM parse failures fall back to `{"message": "Sorry, I encountered an error. Please try again.", "trades": [], "watchlist_changes": []}`

### Concurrency
- `aiosqlite` runs on a thread pool — avoid long transactions
- Price cache reads are lock-free (read-heavy, write-rare)
- Portfolio snapshot task uses `asyncio.sleep` — it's fine to miss a beat under load

### Security
- No user auth (single-user, simulated money)
- Ticker inputs sanitized to uppercase alphanumeric before DB or market source interaction: `re.sub(r'[^A-Z0-9.]', '', ticker.upper())`
- No raw SQL string interpolation — parameterized queries (`?` placeholders) throughout

### Avoiding regressions in market/ code
- Do not modify any files under `backend/app/market/` — that subsystem is complete and reviewed
- Import only the public API: `from app.market import PriceCache, create_market_data_source`

---

## 9. Definition of Done

| Criterion | Check |
|-----------|-------|
| `docker compose up` → `http://localhost:8000` loads the trading terminal | ✓ |
| 10 tickers stream live, prices flash on change, sparklines fill progressively | ✓ |
| Buy/sell trade executes, cash and position update immediately | ✓ |
| Portfolio heatmap and P&L chart render with live data | ✓ |
| AI chat responds, executes trade, confirms in message | ✓ |
| All backend unit tests pass (`uv run pytest`) | ✓ |
| All E2E tests pass with `LLM_MOCK=true` | ✓ |
| Fresh Docker volume starts with clean seeded database | ✓ |
