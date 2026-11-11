# KingStop Architecture

## Frontend

KingStop uses Vite, React 18, TypeScript, Tailwind, Radix UI primitives, Framer Motion, Recharts, and a small Axios API layer. The active entrypoint is `frontend/src/main.tsx`; legacy `.jsx` files still exist but are not the active app shell.

Routing is centralized under `frontend/src/routes`:

- `paths.ts` owns route constants, dashboard section IDs, labels, and type guards.
- `config.tsx` owns lazy page imports and route access metadata.
- `guards.tsx` owns auth-required and public-only route wrappers.

Dashboard sections are URL-driven: `/dashboard/:section`. `/dashboard` redirects to `/dashboard/overview`, invalid dashboard sections route to `/404`, and login preserves the originally requested protected URL through `location.state.from`.

The dashboard is still a large orchestration component. New work should split it by feature domain in this order: `features/dashboard-shell`, `features/predict`, `features/wallet`, `features/account`, `features/portfolio`, and `features/market-news`. Keep route state and cross-feature navigation in the shell; move API calls and local UI state into feature hooks as each section is extracted.

## State And Data

Auth state is in `frontend/src/lib/auth-context.tsx` and persisted through `kingstop_token` / `kingstop_user`. API calls use `frontend/src/services/api.ts`, which attaches the bearer token and redirects to `/login` on 401.

Dashboard section state is now derived from the URL. Section scroll positions are stored in `sessionStorage` per dashboard section. Account preferences are stored server-side through `/api/v1/account/preferences`, with local storage used only as a UI fallback while data loads.

## Backend

The backend is FastAPI with SQLAlchemy and SQLite by default. `backend/main.py` registers routers under `/api/v1`. Database initialization in `backend/database.py` imports model modules and uses `Base.metadata.create_all`.

Core domains:

`main.py` registers the following 27 prefixed routers plus the WebSocket router. Grouped by
concern, not by registration order (per-endpoint details, example payloads, and error codes
live in [docs/api-reference.md](docs/api-reference.md)):

- **Identity** — `auth`: JWT login/register/profile. `account`: profile settings,
  notification and privacy preferences, self-set limits, password changes. `referral`.
- **Market data** — `stocks`, `data`, `news`, `index-compare`, `price-targets`, `watchlist`,
  `screener`, `marketplace`.
- **Analysis** — `signals`, `analytics`, `trading-tools`, `trading-models` (`/models`),
  `ai-explain`, `llm-chat`.
- **Portfolio and trading** — `portfolio`: paper portfolios, wallet, trade execution,
  backtests. `broker`. `journal`. `leaderboard`.
- **Prediction markets** — `predict`: CPMM prediction markets and positions.
- **Alerts and messaging** — `alerts`, `notifications`, `modes`, `social`, `engagement`.
- **Billing** — `payments`: Stripe Checkout integration, gated by environment variables.
- **Realtime** — `websocket` at `/ws/{client_id}`, mounted outside `/api/v1`.

Supporting engines live outside the router layer: `signals/` (technical indicators and an ML
model), `risk/manager.py`, `backtesting/engine.py`, `fusion/engine.py`, `alerts/engine.py`,
`collectors/` (Yahoo Finance ingestion), and `services/`. SQLAlchemy models sit in `models/`.

Stripe keys must remain environment variables (`STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_ENABLED`). No keys should be committed.

## Verification

Primary frontend checks:

- `npm run build`
- `npm run typecheck`
- `npm test`

Backend checks used in this refactor:

- `python -m py_compile api/account.py api/portfolio.py main.py models/user_preferences.py`
- API smoke checks for login, account preferences, password change, prediction buy, and paper trade.

Keep changes staged by domain and commit only after the relevant verification gates pass. Frontend feature work should normally pass type-checking, unit tests, and the production build; backend changes should at minimum pass syntax compilation and targeted API smoke checks.
