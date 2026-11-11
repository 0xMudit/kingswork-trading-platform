<div align="center">

<h1>KingStop</h1>

**An open-source trading-intelligence and paper-trading platform that runs on your machine.**
Live market views, technical and ML signals, paper portfolios, risk management, backtesting,
prediction markets, social trading, and AI-assisted research in one authenticated web app
(colloquially *KingsWork*).

<p>
<a href="https://github.com/0xMudit/kingswork-trading-platform/actions/workflows/ci.yml"><img src="https://github.com/0xMudit/kingswork-trading-platform/actions/workflows/ci.yml/badge.svg" alt="CI status" /></a>
<a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT license" /></a>
<img src="https://img.shields.io/badge/Python-3.11%2B-blue.svg" alt="Python 3.11 or newer" />
<img src="https://img.shields.io/badge/React-18-61dafb.svg" alt="React 18" />
<img src="https://img.shields.io/badge/TypeScript-5-3178c6.svg" alt="TypeScript 5" />
<a href="https://kingswork-ruddy.vercel.app"><img src="https://img.shields.io/badge/live%20demo-Vercel-000000.svg" alt="Live demo" /></a>
<a href="CONTRIBUTING.md"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs welcome" /></a>
</p>

<img src="assets/screenshots/10-dashboard-overview.png" alt="The KingStop dashboard overview: portfolios, signals, and activity in one URL-driven workspace" width="880" />

</div>

---

> **Simulation notice.** KingStop is an engineering and educational project. It is **not financial
> advice** and it never executes real brokerage trades — everything happens in paper portfolios.
> Optional integrations (Yahoo Finance, Redis, Stripe, Groq) are **off by default**, so a fresh clone
> runs against SQLite and deterministic market fixtures with no external services or API keys.

## Why KingStop

Trading research is scattered across brokers, charting tools, news tickers, chat groups, and
spreadsheets. KingStop closes the loop on your own machine: every page reads from the same FastAPI
`/api/v1` surface that the signal engines, risk manager, and backtester write to — one coherent view
of the whole research-to-trade loop, and a safe place to rehearse it.

- **Self-contained.** One `python` + `npm` pair boots the whole platform against SQLite and
  deterministic market fixtures. No brokerage, exchange, or API key required.
- **Real markets, optionally.** Yahoo Finance can stream daily data for US, NSE, BSE, and crypto
  tickers; the simulated collectors keep every feature demoable offline.
- **Transparent models.** Signals show their indicators, the risk manager reports position sizing,
  value-at-risk, and exposure, backtests report fills, commission, PnL, and drawdown, and prediction
  markets resolve with auditable CPMM math.
- **Safe by construction.** Stop-loss / take-profit defaults, position caps, and responsible-use
  guardrails ship enabled.
- **Social by default.** Share research through portfolios, leaderboards, feeds, streaks, and daily
  challenges — without handing over your keys.
- **One API surface.** 28 domain routers under a versioned `/api/v1`, documented interactively at
  `/docs` and paired with WebSocket channels.

## Features

| Area | What you get |
| --- | --- |
| Market data | US / NSE / BSE / crypto price views, historical and real-time workflows, index comparisons, and watchlists. |
| Signals & screening | Technical and ML signal engines, screeners, news, price targets, and prediction-style intelligence. |
| Portfolio & trading | User-owned paper portfolios, wallet, positions, trade plans, trade copy, and backtests. |
| Risk | Exposure, correlation matrix, market heatmap, stop-loss / take-profit defaults, position and risk caps. |
| Prediction markets | CPMM markets with positions, streaks, achievements, and daily challenges. |
| Social | Trading journal, social feed, market header, referrals, alerts, and notifications. |
| AI research | Groq-powered market assistant with the in-app KingStop chat personality. |
| Intelligence stack | Fusion engine, trading-models layer, AI explain, and a developer reference. |
| Identity | JWT bearer auth, bcrypt hashing, account preferences, security settings, and guided onboarding. |
| Realtime | WebSocket channels at `/ws/{client_id}` for live update workflows. |
| Developer surface | Versioned `/api/v1` FastAPI API with OpenAPI docs and 28 domain routers. |
| Verification | Backend risk/backtest suites plus frontend typecheck, tests, and build — all CI-gated. |

## Screenshots

**Dashboard overview** — the URL-driven center of the app: portfolios, signals, and activity at a
glance.

![Dashboard overview](assets/screenshots/10-dashboard-overview.png)

**Market header** — a live US / NSE / BSE / crypto snapshot with index comparison.

![Market header](assets/screenshots/08-market-header.png)

**Signals marketplace** — technical and ML signals browsable like a marketplace.

![Signals marketplace](assets/screenshots/04-signals-marketplace.png)

**Prediction markets** — CPMM markets with auditable resolution.

![Prediction markets](assets/screenshots/05-prediction-markets.png)

**Portfolio leaderboard** — community standings, streaks, and achievements.

![Portfolio leaderboard](assets/screenshots/01-portfolio-leaderboard.png)

**Groq-powered chat** — fast model responses when `GROQ_API_KEY` is configured.

![Groq-powered chat](assets/screenshots/11-groq-powered-chat.png)

**Security settings** — password, preferences, notifications, privacy, and self-set limits.

![Security settings](assets/screenshots/02-security-settings.png)

Full-size captures live in [`assets/screenshots/`](./assets/screenshots/).

## Requirements

- **Python 3.11+**
- **Node.js 20+** with npm 9+
- **Docker** (optional) for the one-command full stack
- **Optional services** — Groq (`GROQ_API_KEY`), Stripe (`STRIPE_*`), and Redis (`USE_REDIS`) are
  all opt-in and ship disabled.

## Quick start

### 1. Start the API

```bash
git clone https://github.com/0xMudit/kingswork-trading-platform.git
cd kingswork-trading-platform/backend
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

The API is available at <http://localhost:8000>, with interactive OpenAPI docs at
<http://localhost:8000/docs>.

### 2. Start the frontend

In a second terminal:

```bash
cd kingswork-trading-platform/frontend
npm install
npm run dev
```

Open <http://localhost:5173/kingswork/>. The Vite dev server proxies `/api` and `/ws` to the local
FastAPI server. The dev database seeds demo users and sample trading data — override all demo and
test account values in `backend/.env` for any shared deployment.

### Docker

```bash
docker compose up --build
```

Starts the API on port `8000` and the frontend on port `5173`. Review the compose environment and
reverse-proxy settings before production use.

## Configuration

Copy `backend/.env.example` to `backend/.env`. The app works locally with SQLite and simulated
market data; external service keys are optional. Configuration is read through `backend/config.py`.

| Group | Important variables |
| --- | --- |
| Core | `DATABASE_URL` · `SECRET_KEY` · `DEBUG` · `API_PREFIX` |
| Market data | `USE_LIVE_MARKET_DATA` · `YAHOO_REFRESH_INTERVAL` · `MARKET_DATA_TIMEOUT_SECONDS` |
| AI chat | `GROQ_API_KEY` · `GROQ_MODEL` · `GROQ_TIMEOUT_SECONDS` |
| Cache | `USE_REDIS` · `REDIS_URL` |
| Billing | `STRIPE_ENABLED` · `STRIPE_SECRET_KEY` · `STRIPE_PUBLISHABLE_KEY` · `STRIPE_WEBHOOK_SECRET` |
| Risk defaults | `MAX_POSITION_SIZE_PCT` · `MAX_PORTFOLIO_RISK_PCT` · `STOP_LOSS_PCT` |

Never commit `backend/.env`, databases, logs, or production credentials — all excluded by
`.gitignore`.

## How it works

```
┌─────────────────────────────────────────── Frontend ───────────────────────────────────────────┐
│  React 18 + Vite SPA (TypeScript, Tailwind, Radix UI, Framer Motion, Recharts, Axios)           │
│  ├─ routes/      path constants, lazy page config, auth guards                                  │
│  ├─ pages/       landing, auth, documentation, dashboard                                        │
│  ├─ features/    navigation, onboarding, chat, feature modules                                  │
│  └─ services/    typed API client (bearer token; redirects to /login on 401)                    │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
            │ REST /api/v1                                        │ WebSocket /ws/{client_id}
            ▼                                                     ▼
┌─────────────────────────────────────────── FastAPI backend ────────────────────────────────────┐
│  main.py — 28 domain routers under /api/v1 plus the WebSocket router                            │
│  ├─ Identity       auth · account · referral                                                    │
│  ├─ Market data    stocks · data · news · index-compare · price-targets · watchlist · screener  │
│  ├─ Analysis       signals · analytics · trading-tools · trading-models · ai-explain · llm-chat  │
│  ├─ Portfolio      portfolio · broker · journal · leaderboard                                   │
│  ├─ Prediction     predict (CPMM markets)                                                       │
│  ├─ Alerts/social  alerts · notify · modes · social · engagement                                │
│  └─ Billing        payments (Stripe, opt-in)                                                    │
│                                                                                                 │
│  Engines: signals/ (technical + ML) · risk/manager.py · backtesting/engine.py ·                 │
│           fusion/engine.py · alerts/engine.py · collectors/ (Yahoo Finance)                     │
│  Data:    SQLAlchemy models → SQLite (configurable DSN) · optional Redis · Yahoo Finance        │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The frontend is a React SPA whose dashboard sections are URL-driven at `/dashboard/:section` and
lazy-loaded; every page talks to the same `/api/v1` surface through a typed Axios client that
attaches the JWT bearer token. On the backend, `main.py` registers 28 domain routers plus a
WebSocket channel, and each domain delegates to a shared engine layer: `signals/` computes technical
indicators and an ML model, `risk/manager.py` does position sizing, risk scoring, and value-at-risk,
`backtesting/engine.py` models fills, commission, PnL, and drawdown, and `fusion/engine.py` combines
signals into the intelligence stack. SQLAlchemy models persist to SQLite by default, with Redis,
Yahoo Finance, Stripe, and Groq all gated behind environment flags. See
[ARCHITECTURE.md](ARCHITECTURE.md) for routing, state ownership, and extension guidance,
and [docs/api-reference.md](docs/api-reference.md) for the per-endpoint API reference.

## Project layout

```
backend/            FastAPI + SQLAlchemy + SQLite
  api/              Versioned domain routers (/api/v1)
  auth/             JWT authentication and request dependencies
  signals/          Technical indicators and ML signal models
  risk/             Position sizing, risk scoring, exposure
  backtesting/      Fill, commission, PnL, and drawdown engine
  fusion/           Multi-signal fusion engine
  alerts/           Alert matching and dispatch
  collectors/       Market data acquisition
  models/           SQLAlchemy domain models
  services/         Shared business logic
  main.py           FastAPI application and route registration
  config.py         Typed environment configuration
frontend/           React 18 + Vite + Tailwind + Recharts
  src/components/   Trading and analytics UI
  src/features/     Navigation, onboarding, chat, feature modules
  src/pages/        Landing, authentication, documentation, dashboard
  src/routes/       Route definitions and guards
  src/services/     Typed API client functions
assets/
  screenshots/      Product screenshots used in this README
docker-compose.yml  Backend + frontend stack
```

## Commands

| Command | Description |
| --- | --- |
| `uvicorn main:app --reload` | Run the API from `backend/` on port 8000. |
| `npm run dev` | Run the Vite dev server from `frontend/` on port 5173. |
| `npm run typecheck` | Type-check the frontend with `tsc --noEmit`. |
| `npm test` | Run the frontend Vitest suite. |
| `npm run build` | Production frontend build. |
| `python -m pytest` | Run the backend test suite from `backend/`. |
| `docker compose up --build` | Boot the full stack (API + frontend). |

The backend suite covers the risk manager (position sizing, risk scoring, value-at-risk) and the
backtest engine (fills, commission, PnL, drawdown). CI runs both suites plus the frontend typecheck,
tests, and build on Node 22 and Python 3.12 for every push and pull request.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Markets look static or "fake" | `USE_LIVE_MARKET_DATA` is `false` by default — that is the simulated-fixture mode. Set it to `true` for Yahoo Finance data (needs network access). |
| AI chat returns an error | The Groq assistant is opt-in. Set `GROQ_API_KEY` (and optionally `GROQ_MODEL`) in `backend/.env` and restart the API. |
| Frontend calls fail or 401 loops | Confirm the API is running on `:8000`; Vite proxies `/api` and `/ws` to it. Sign in with seeded demo credentials or register a fresh account. |
| Redis or Stripe errors on boot | Both are opt-in. Leave `USE_REDIS=false` and `STRIPE_ENABLED=false`, or provide the matching keys. |
| `source .venv/bin/activate` fails on Windows | Use `.venv\Scripts\activate` in PowerShell instead. |
| Database state looks stale | The dev database is seeded on first boot; delete the local SQLite file to re-seed. |

## Roadmap

The [ROADMAP.md](ROADMAP.md) tracks proposed future work beyond the current implementation. Items are
drafts pending owner confirmation, but the shape of it:

- **Phase 1** — pluggable broker adapters and real order execution (paper stays the default).
- **Phase 2** — additional market-data providers and intraday WebSocket price streaming.
- **Phase 3** — an open, trainable ML signal layer with a feature store and model registry.
- **Phase 4** — realtime social feeds, comments, reactions, and moderation tooling.
- **Phase 5** — Progressive Web App, responsive audit, and accessibility passes.
- **Phase 6** — Kubernetes manifests, Prometheus/Grafana, tracing, and load testing.

## Contributing

Contributions are welcome — issues, docs, and pull requests alike. Start with
[CONTRIBUTING.md](CONTRIBUTING.md) for setup, code style, and the reviewable-PR bar, and
[ARCHITECTURE.md](ARCHITECTURE.md) for where the code lives. Roadmap work is tracked in
[ROADMAP.md](ROADMAP.md). Please read the [Code of Conduct](CODE_OF_CONDUCT.md); it applies to every
project space.

Good first issues are labelled
[`good first issue`](https://github.com/0xMudit/kingswork-trading-platform/labels/good%20first%20issue).

## Security

KingStop is a paper-trading and research platform and must never be connected to a real brokerage.
Keys and credentials (`STRIPE_*`, `GROQ_API_KEY`, `SECRET_KEY`) must remain environment variables —
never commit `backend/.env`, databases, logs, or production secrets. Report vulnerabilities privately
per [SECURITY.md](SECURITY.md) rather than in a public issue.

## License

[MIT](LICENSE) © 2026 Muditya Raghav — see the LICENSE file for details.
