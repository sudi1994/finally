# FinAlly — AI Trading Workstation

> Your Finance Ally: a visually stunning AI-powered trading workstation that streams live market data, lets you trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on your behalf.

Built entirely by coding agents as a capstone project for an agentic AI coding course — demonstrating how orchestrated AI agents can produce a production-quality full-stack application.

![Dark terminal aesthetic inspired by Bloomberg](https://img.shields.io/badge/aesthetic-Bloomberg%20terminal-0d1117?style=flat-square&labelColor=209dd7&color=0d1117)
![FastAPI](https://img.shields.io/badge/backend-FastAPI-009688?style=flat-square)
![Next.js](https://img.shields.io/badge/frontend-Next.js-000000?style=flat-square&logo=next.js)
![SQLite](https://img.shields.io/badge/database-SQLite-003B57?style=flat-square&logo=sqlite)

---

## What Is FinAlly?

FinAlly is a single-page trading terminal you run locally in Docker. On first launch you see:

- A **watchlist** of 10 default tickers (AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX) with live-updating prices
- **$10,000 in virtual cash** — no real money, no risk
- A **dark, data-rich terminal aesthetic** inspired by Bloomberg
- An **AI chat panel** ready to analyze your portfolio and execute trades via natural language

No login. No signup. One command and you're trading.

---

## Features

### Live Market Data
- Prices stream in real-time via **Server-Sent Events (SSE)**
- **Green/red flash animations** on price upticks/downticks, fading over ~500ms
- **Sparkline mini-charts** accumulate live price action beside each ticker
- **Connection status indicator** in the header (green = connected, yellow = reconnecting, red = disconnected)
- Default: built-in **GBM price simulator** — no API key required
- Optional: **Massive (Polygon.io) API** for real market data

### Portfolio & Trading
- **Market orders only** — instant fill at current price, no fees, no confirmation
- **$10,000 starting cash** — fractional shares supported
- **Positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Portfolio heatmap** — treemap showing positions sized by portfolio weight, colored by P&L
- **P&L chart** — line chart tracking total portfolio value over time
- **Trade bar** — ticker input, quantity input, Buy / Sell buttons

### AI Chat Assistant
- Powered by **LiteLLM → OpenRouter (Cerebras inference)** for fast responses
- Understands your full portfolio context on every message
- **Auto-executes trades** — tell it "buy 10 shares of NVDA" and it happens instantly
- **Manages your watchlist** — add or remove tickers via natural language
- Full conversation history persisted in SQLite
- Trade confirmations shown inline in the chat

### Watchlist Management
- Add/remove tickers manually or ask the AI
- Prices update live for all watched tickers
- Click any ticker to view a larger detailed chart in the main chart area

---

## Architecture

Everything runs in a **single Docker container on port 8000**.

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim       │
└─────────────────────────────────────────────────┘
```

| Layer | Technology | Notes |
|---|---|---|
| Frontend | Next.js + TypeScript + Tailwind CSS | Static export, served by FastAPI |
| Backend | FastAPI (Python 3.12, managed with `uv`) | REST + SSE endpoints |
| Database | SQLite | Lazy-initialized, volume-mounted for persistence |
| AI | LiteLLM → OpenRouter (Cerebras) | Structured JSON output, fast inference |
| Market data | GBM simulator (default) or Massive REST API | Swapped via env var |
| Real-time | Server-Sent Events (SSE) | One-way push, universal browser support |

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need — simpler, no bidirectional complexity |
| Static Next.js export | Single origin, no CORS, one port, one container |
| SQLite over Postgres | No auth = no multi-user = no need for a database server |
| Single Docker container | One command to run; no docker-compose for end users |
| `uv` for Python | Fast, modern dependency management with reproducible lockfile |
| Market orders only | Eliminates order book complexity; dramatically simpler portfolio math |

---

## Quick Start

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- An [OpenRouter](https://openrouter.ai/) API key (for AI chat)

### 1. Clone and configure

```bash
git clone https://github.com/sudi1994/finally.git
cd finally
cp .env.example .env
```

Edit `.env` and add your OpenRouter API key:

```bash
OPENROUTER_API_KEY=your-openrouter-api-key-here
```

### 2. Build and run

```bash
docker build -t finally .
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

### 3. Open the app

Navigate to **http://localhost:8000** in your browser.

### Stop the container

```bash
docker stop $(docker ps -q --filter ancestor=finally)
```

> **Note:** Your portfolio data persists in the `finally-data` Docker volume across container restarts. To reset to a fresh state: `docker volume rm finally-data`

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `OPENROUTER_API_KEY` | **Yes** | — | OpenRouter API key for AI chat functionality |
| `MASSIVE_API_KEY` | No | — | Massive (Polygon.io) key for real market data; omit to use the built-in simulator |
| `LLM_MOCK` | No | `false` | Set `true` for deterministic mock LLM responses (used in E2E tests) |

### Market Data Modes

- **Simulator (default)** — no API key needed. Generates realistic prices using Geometric Brownian Motion with correlated moves across tickers and occasional random "events" (sudden 2–5% moves). Updates every ~500ms.
- **Massive API** — set `MASSIVE_API_KEY` to poll real market prices. Free tier supports polling every ~15 seconds; paid tiers support 2–15 second intervals.

---

## API Reference

### Market Data
| Method | Path | Description |
|---|---|---|
| `GET` | `/api/stream/prices` | SSE stream of live price updates for all watched tickers |

### Portfolio
| Method | Path | Description |
|---|---|---|
| `GET` | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| `POST` | `/api/portfolio/trade` | Execute a trade: `{"ticker": "AAPL", "quantity": 10, "side": "buy"}` |
| `GET` | `/api/portfolio/history` | Portfolio value snapshots over time (for the P&L chart) |

### Watchlist
| Method | Path | Description |
|---|---|---|
| `GET` | `/api/watchlist` | Current watchlist tickers with latest prices |
| `POST` | `/api/watchlist` | Add a ticker: `{"ticker": "AAPL"}` |
| `DELETE` | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|---|---|---|
| `POST` | `/api/chat` | Send a message, receive AI response + any executed trades/watchlist changes |

### System
| Method | Path | Description |
|---|---|---|
| `GET` | `/api/health` | Health check |

---

## Project Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   ├── app/                  # Application code (routes, models, services)
│   └── db/                   # Schema definitions, seed data
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # Full project specification
│   └── MARKET_DATA_SUMMARY.md
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
├── scripts/                  # Start/stop helper scripts
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env.example              # Template for environment variables
└── .gitignore
```

---

## Database Schema

The SQLite database is **lazily initialized** — created automatically on first run with default seed data. No manual migration step required.

| Table | Purpose |
|---|---|
| `users_profile` | Cash balance per user |
| `watchlist` | Tickers being watched |
| `positions` | Current holdings (quantity, average cost) |
| `trades` | Append-only trade history log |
| `portfolio_snapshots` | Portfolio value over time (recorded every 30s + after each trade) |
| `chat_messages` | Full conversation history with the AI assistant |

Default seed: one user with `$10,000` cash and 10 default watchlist tickers.

---

## AI Chat Examples

Ask FinAlly anything about your portfolio:

```
"What's my current portfolio allocation?"
"Buy 5 shares of AAPL"
"Sell all my TSLA position"
"Add PYPL to my watchlist"
"I'm too concentrated in tech — what should I do?"
"Show me my biggest winners and losers"
```

The AI has full context of your positions, cash balance, live prices, and conversation history on every message. Trades execute immediately — no confirmation required.

---

## Development

### Running locally without Docker

**Backend:**
```bash
cd backend
uv sync
uv run uvicorn app.main:app --reload --port 8000
```

**Frontend:**
```bash
cd frontend
npm install
npm run dev
```

### Running tests

**Backend unit tests:**
```bash
cd backend
uv run pytest
```

**E2E tests (requires Docker):**
```bash
cd test
docker compose -f docker-compose.test.yml up --abort-on-container-exit
```

---

## Visual Design

- **Color palette**: Dark backgrounds (`#0d1117` / `#1a1a2e`), accent yellow (`#ecad0a`), blue primary (`#209dd7`), purple secondary (`#753991`)
- **Price flash**: Brief green/red background highlight on each price change, CSS transition fade over ~500ms
- **Desktop-first**: Optimized for wide screens, functional on tablet

---

## License

[MIT License](LICENSE) — Copyright (c) 2026 Ed Donner
