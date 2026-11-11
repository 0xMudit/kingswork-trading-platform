# API Reference

Per-endpoint reference for the KingStop backend. [ARCHITECTURE.md](../ARCHITECTURE.md)
lists the domain routers and where they live; this document describes what each
endpoint accepts and returns. It is generated from the routers registered in
`backend/main.py` — if you add or change an endpoint, update this file in the
same PR.

## Conventions

- **Base URL:** all REST routers are mounted under `/api/v1`
  (`settings.api_prefix`). The WebSocket router is the only exception; it is
  mounted at `/ws/{client_id}` outside `/api/v1`.
- **Auth:** endpoints marked 🔒 require a bearer token:
  `Authorization: Bearer <access_token>`, obtained from `POST /api/v1/auth/login`.
  Endpoints without the marker take no user dependency in their handler.
- **Responses:** most handlers return a JSON envelope with `"status": "ok"`
  plus a domain payload key, e.g. `{"status": "ok", "portfolio": {...}}`.
  Market-data endpoints (`/stocks`, `/signals`) return
  `{"status": "ok", "data": ...}`.
- **Errors:** FastAPI's standard `{"detail": "..."}` body. Codes used across
  the API:

  | Code | Meaning |
  |------|---------|
  | 400 | Bad request (invalid input, insufficient funds/liquidity, bad state transition) |
  | 401 | Missing/invalid bearer token, or invalid login credentials |
  | 403 | Feature gated for the current plan/mode |
  | 404 | Referenced resource (portfolio, market, rule, entry, …) not found |
  | 409 | Conflict (duplicate watchlist ticker, already-published strategy, …) |
  | 422 | Request validation failed (missing/invalid fields) |
  | 500 | Signal/market-data engine failure |
  | 502 | Upstream index/data provider failure |
  | 503 | Payment provider not configured/available |

## Auth — `backend/auth/router.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/register` | — | Create an account |
| POST | `/api/v1/auth/login` | — | Log in, returns a bearer token |
| GET | `/api/v1/auth/me` | 🔒 | Current user profile |

`POST /api/v1/auth/register` — body (`RegisterRequest`):

```json
{ "email": "trader@example.com", "password": "s3cret-pass", "name": "Trader One" }
```

Returns the created user (`UserResponse`): `{ "id": 1, "email": "trader@example.com", "name": "Trader One", "is_active": true }`.
Errors: `400` if the email is already registered or the password is rejected.

`POST /api/v1/auth/login` — form-encoded (`OAuth2PasswordRequestForm`):
`username=<email>&password=<password>`. Returns:

```json
{
  "access_token": "<jwt>",
  "token_type": "bearer",
  "user": { "id": 1, "email": "trader@example.com", "name": "Trader One" },
  "streak": { "current_streak": 3, "longest_streak": 9, "total_logins": 12, "bonus_claimed_today": false, "is_new_day": true }
}
```

Errors: `401` — `"Invalid email or password"`.

## Account — `backend/api/account.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/account/settings` | 🔒 | Account settings |
| GET | `/api/v1/account/onboarding` | 🔒 | Onboarding state |
| PUT | `/api/v1/account/onboarding` | 🔒 | Update onboarding state |
| PUT | `/api/v1/account/first-action` | 🔒 | Record the first completed action |
| PUT | `/api/v1/account/preferences` | 🔒 | Update account preferences |
| POST | `/api/v1/account/security/password` | 🔒 | Change password |

`PUT /api/v1/account/preferences` — body (`AccountPreferencesRequest`, all fields optional):

```json
{ "leaderboardOptIn": true, "priceAlerts": true, "dailyFundingLimit": 500.0, "exposureLimit": 2500.0 }
```

`POST /api/v1/account/security/password` — body (`PasswordChangeRequest`):

```json
{ "current_password": "old-pass", "new_password": "new-pass" }
```

Errors: `400` when the current password does not match.

## Stocks — `backend/api/stocks.py`

Market data. All endpoints return `{"status": "ok", "data": ...}` and may
return `500` when the upstream data engine fails.

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/stocks/realtime/{ticker}` | — | — |
| GET | `/api/v1/stocks/historical/{ticker}` | — | `period`, `interval` (query) |
| GET | `/api/v1/stocks/overview` | — | `tickers` (comma-separated, query), `market` (query) |
| GET | `/api/v1/stocks/analysis/{ticker}` | — | `period`, `model_id` (query) |

Example — `GET /api/v1/stocks/realtime/AAPL`:

```json
{ "status": "ok", "data": { "ticker": "AAPL", "price": 231.42, "change_pct": 1.8 } }
```

## Signals — `backend/api/signals.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/signals/{ticker}` | — | `period`, `model_id` (query) |
| GET | `/api/v1/signals/all/current` | — | — |

Example — `GET /api/v1/signals/AAPL?period=6mo`:

```json
{ "status": "ok", "data": { "ticker": "AAPL", "signal": "buy", "confidence": 0.72 } }
```

Errors: `500` when the signal engine cannot produce a result.

## AI explain — `backend/api/ai_explain.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/ai-explain/{ticker}` | 🔒 | `period`, `model_id` (query) |

Returns a natural-language explanation of the current signal/analysis for the
ticker, in the same `{"status": "ok", "data": ...}` envelope.

## LLM chat — `backend/api/llm_chat.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/llm-chat` | 🔒 | Chat completion over the trading assistant |

Body (`ChatRequest`):

```json
{ "messages": [ { "role": "user", "content": "Why is AAPL trending up today?" } ] }
```

`role` is one of `user` / `assistant`.

## Portfolio — `backend/api/portfolio.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/portfolio/portfolios` | 🔒 | List my portfolios |
| POST | `/api/v1/portfolio/portfolios` | 🔒 | Create a portfolio |
| GET | `/api/v1/portfolio/portfolios/{portfolio_id}` | 🔒 | One portfolio with positions |
| POST | `/api/v1/portfolio/portfolios/{portfolio_id}/trade` | 🔒 | Execute a paper trade |
| GET | `/api/v1/portfolio/wallet` | 🔒 | Demo wallet |
| GET | `/api/v1/portfolio/backtest/{ticker}` | — | Backtest one ticker (`period`, `model_id` query) |
| POST | `/api/v1/portfolio/paper` | 🔒 | Create a paper portfolio |
| GET | `/api/v1/portfolio/paper` | 🔒 | List paper portfolios |
| GET | `/api/v1/portfolio/paper/summary` | 🔒 | Paper-trading summary |
| GET | `/api/v1/portfolio/pnl-live` | 🔒 | Live P&L across portfolios |

`POST /api/v1/portfolio/portfolios` — body (`PortfolioCreate`):

```json
{ "name": "Long-term", "initial_cash": 10000.0 }
```

Returns `{ "status": "ok", "portfolio": { "id": 3, "name": "Long-term", "cash": 10000.0, "positions": [] } }`.

`POST /api/v1/portfolio/portfolios/{portfolio_id}/trade` — body (`TradeRequest`):

```json
{ "ticker": "AAPL", "side": "buy", "quantity": 5, "price": 231.42, "order_type": "market" }
```

`side` is an `OrderSide` (`buy`/`sell`); `order_type` is an `OrderType`
(`market`/`limit`). Errors: `400` on insufficient cash/shares, `404` for an
unknown portfolio.

## Analytics — `backend/api/analytics.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/analytics/pnl/{portfolio_id}` | 🔒 | P&L history |
| GET | `/api/v1/analytics/sector-exposure/{portfolio_id}` | 🔒 | Sector exposure |
| GET | `/api/v1/analytics/overview/{portfolio_id}` | 🔒 | Performance overview |
| GET | `/api/v1/analytics/correlation` | — | `tickers` (comma-separated), `period` (query) |
| GET | `/api/v1/analytics/options-chain/{ticker}` | — | `expiry` (query, optional) |

Example — `GET /api/v1/analytics/pnl/3` returns
`{ "status": "ok", "history": [ { "date": "2026-01-05", "value": 10231.5 } ] }`-style
series for the portfolio. Errors: `404` for an unknown portfolio.

## Index compare — `backend/api/index_compare.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/index-compare/{portfolio_id}` | 🔒 | `index` (query, e.g. `SPX`) |

Errors: `400` for an unsupported index, `404` for an unknown portfolio,
`502` when the index provider fails.

## Predict (prediction markets) — `backend/api/predict.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/predict` | 🔒 | List markets (`category`, `status` query filters) |
| GET | `/api/v1/predict/{market_id}` | 🔒 | One market |
| POST | `/api/v1/predict` | 🔒 | Create a market |
| GET | `/api/v1/predict/{market_id}/quote` | 🔒 | Quote a buy (`side`, `cost` query) |
| POST | `/api/v1/predict/{market_id}/buy` | 🔒 | Buy shares |
| POST | `/api/v1/predict/{market_id}/sell` | 🔒 | Sell shares |
| POST | `/api/v1/predict/{market_id}/resolve` | 🔒 | Resolve a market (`outcome` query) |

`POST /api/v1/predict` — body (`CreateMarketRequest`):

```json
{
  "question": "Will AAPL close above $250 this month?",
  "category": "stocks",
  "ticker": "AAPL",
  "condition": "close_above",
  "threshold": 250.0,
  "initial_probability": 0.5,
  "liquidity": 1000.0
}
```

Returns `{ "status": "ok", "market": { "id": 7, "question": "...", "probability": 0.5 } }`.

`POST /api/v1/predict/{market_id}/buy` — body (`BuyRequest`): `{ "side": "yes", "cost": 25.0 }`.
`POST /api/v1/predict/{market_id}/sell` — body (`SellRequest`): `{ "position_id": 11, "shares": 10.0 }`.

Errors: `400` on insufficient funds/liquidity or an invalid side, `404` for an
unknown market.

## Payments — `backend/api/payments.py`

Stripe-backed purchases of in-app cash and subscriptions.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/payments/buy-cash` | 🔒 | Start a cash purchase checkout |
| POST | `/api/v1/payments/subscribe` | 🔒 | Start a subscription checkout |
| GET | `/api/v1/payments/subscription` | 🔒 | Current subscription |
| POST | `/api/v1/payments/cancel-subscription` | 🔒 | Cancel the subscription |
| POST | `/api/v1/payments/webhook` | — | Stripe webhook (signature-verified) |
| GET | `/api/v1/payments/history` | 🔒 | Payment history |

`POST /api/v1/payments/buy-cash` — body (`BuyCashRequest`): `{ "amount_usd": 100.0 }`.
Returns a checkout session reference for the client to complete payment.

`POST /api/v1/payments/subscribe` — body (`SubscribeRequest`): `{ "tier": "pro", "billing": "monthly" }`.

`POST /api/v1/payments/webhook` — raw Stripe event body with the
`Stripe-Signature` header; not called by clients directly.

Errors: `400` for an invalid amount/tier, `404` when there is no subscription
to cancel, `503` when Stripe is not configured.

## Alerts — `backend/api/alerts_api.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/alerts/rules` | List alert rules |
| POST | `/api/v1/alerts/rules` | Create an alert rule |
| DELETE | `/api/v1/alerts/rules/{rule_id}` | Delete a rule |
| GET | `/api/v1/alerts/history` | Alert history (`ticker`, `limit` query) |
| POST | `/api/v1/alerts/{alert_id}/acknowledge` | Acknowledge an alert |

`POST /api/v1/alerts/rules` — body (`RuleCreate`):

```json
{ "name": "AAPL dip", "ticker": "AAPL", "condition": "price_below", "threshold": 220.0, "severity": "high", "enabled": true }
```

`condition` is an `AlertCondition` and `severity` an `AlertSeverity` enum from
that module. Errors: `404` for an unknown rule/alert id.

## Data (earnings, insider trades, FII/DII, SMS) — `backend/api/data_alerts.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/data/earnings` | — | Earnings calendar (`ticker` query, optional) |
| GET | `/api/v1/data/insider-trades` | — | Insider trades (`ticker` query, optional) |
| GET | `/api/v1/data/fii-dii` | — | FII/DII flow data |
| POST | `/api/v1/data/sms-config` | 🔒 | Save SMS/WhatsApp alert config |
| GET | `/api/v1/data/sms-config` | 🔒 | Read SMS/WhatsApp alert config |

`POST /api/v1/data/sms-config` — body (`SMSConfig`):

```json
{ "phone": "+15555550100", "whatsapp": "+15555550100", "enabled": true }
```

## Broker — `backend/api/broker.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/broker/connect` | 🔒 | Connect a broker account |
| GET | `/api/v1/broker/connections` | 🔒 | List connections |
| DELETE | `/api/v1/broker/connections/{conn_id}` | 🔒 | Disconnect |
| POST | `/api/v1/broker/order` | 🔒 | Place a broker order |
| GET | `/api/v1/broker/orders` | 🔒 | List broker orders |

`POST /api/v1/broker/connect` — body (`ConnectIn`):

```json
{ "broker": "zerodha", "api_key": "...", "api_secret": "..." }
```

`POST /api/v1/broker/order` — body (`OrderIn`):

```json
{ "broker": "zerodha", "ticker": "RELIANCE", "side": "buy", "qty": 10, "price": 2450.0, "order_type": "limit" }
```

## Watchlist — `backend/api/watchlist.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/watchlist` | 🔒 | My watchlist |
| POST | `/api/v1/watchlist/{ticker}` | 🔒 | Add a ticker |
| PUT | `/api/v1/watchlist/{ticker}` | 🔒 | Update a watchlist entry |
| DELETE | `/api/v1/watchlist/{ticker}` | 🔒 | Remove a ticker |

`PUT /api/v1/watchlist/{ticker}` — body (`WatchlistUpdate`): `{ "ticker": "AAPL" }`.

Errors: `201` on add, `404` for an unknown ticker, `409` if the ticker is
already on the watchlist, `422` for an invalid payload.

## Screener — `backend/api/screener.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/screener` | — | `market`, `signal`, `min_change_pct`, `max_change_pct`, `min_volume`, `sort_by`, `order` (query) |

Example — `GET /api/v1/screener?market=US&min_change_pct=2&sort_by=change_pct&order=desc`
returns the matching stocks in the `{"status": "ok", "data": [...]}` envelope.

## News — `backend/api/news.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/news/{ticker}` | — | `limit` (query) |

Returns recent news items for the ticker in the `{"status": "ok", "data": [...]}`
envelope.

## Journal — `backend/api/journal.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/journal` | 🔒 | List entries (`ticker` query, optional) |
| POST | `/api/v1/journal` | 🔒 | Create an entry |
| DELETE | `/api/v1/journal/{entry_id}` | 🔒 | Delete an entry |

`POST /api/v1/journal` — body (`JournalCreate`):

```json
{ "ticker": "AAPL", "note": "Bought the dip after earnings", "sentiment": "bullish", "entry_price": 220.5, "exit_price": null, "pnl": null, "tags": "earnings" }
```

Errors: `404` for an unknown entry id.

## Leaderboard — `backend/api/leaderboard.py`

| Method | Path | Auth | Key params |
|--------|------|------|------------|
| GET | `/api/v1/leaderboard` | 🔒 | `page`, `page_size` (query) |
| GET | `/api/v1/leaderboard/stats/{portfolio_id}` | 🔒 | Stats for one portfolio |

Errors: `404` for an unknown portfolio.

## Social — `backend/api/social.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/social/follow/{user_id}` | 🔒 | Follow a user |
| GET | `/api/v1/social/following` | 🔒 | Who I follow |
| GET | `/api/v1/social/followers` | 🔒 | My followers |
| GET | `/api/v1/social/ideas` | — | List trade ideas (`ticker` query, optional) |
| POST | `/api/v1/social/ideas` | 🔒 | Post a trade idea |
| POST | `/api/v1/social/ideas/{idea_id}/like` | 🔒 | Like an idea |
| POST | `/api/v1/social/ideas/{idea_id}/comments` | 🔒 | Comment on an idea |
| GET | `/api/v1/social/ideas/{idea_id}/comments` | — | List comments |
| GET | `/api/v1/social/weekly-badges` | — | This week's badges |

`POST /api/v1/social/ideas` — body (`IdeaCreate`):

```json
{ "ticker": "AAPL", "direction": "long", "entry": 228.0, "target": 250.0, "stop": 215.0, "body": "Breaking out on volume." }
```

`POST /api/v1/social/ideas/{idea_id}/comments` — body (`CommentCreate`): `{ "body": "Agree — watching the same level." }`.

## Price targets & trade copy — `backend/api/price_targets.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/price-targets` | 🔒 | Create a price target |
| GET | `/api/v1/price-targets` | 🔒 | List my price targets |
| DELETE | `/api/v1/price-targets/{target_id}` | 🔒 | Delete a price target |
| POST | `/api/v1/trade-copy/follow` | 🔒 | Follow a portfolio |
| GET | `/api/v1/trade-copy/following` | 🔒 | Portfolios I follow |
| DELETE | `/api/v1/trade-copy/{follow_id}` | 🔒 | Unfollow |
| GET | `/api/v1/trade-copy/feed` | 🔒 | Trade feed from followed portfolios |

`POST /api/v1/price-targets` — body (`PriceTargetCreate`):

```json
{ "ticker": "AAPL", "target_price": 260.0, "direction": "above", "note": "Post-earnings run" }
```

`POST /api/v1/trade-copy/follow` — body (`FollowRequest`): `{ "portfolio_id": 3 }`.

Errors: `201` on create, `400` for an invalid target/follow, `404` for an
unknown id, `409` on duplicates.

## Marketplace — `backend/api/marketplace.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/marketplace` | — | List published strategies |
| POST | `/api/v1/marketplace` | 🔒 | Publish a strategy |
| GET | `/api/v1/marketplace/my/published` | 🔒 | My published strategies |
| GET | `/api/v1/marketplace/my/purchased` | 🔒 | My purchased strategies |
| GET | `/api/v1/marketplace/{strategy_id}` | — | One strategy |
| POST | `/api/v1/marketplace/{strategy_id}/buy` | 🔒 | Buy a strategy |

`POST /api/v1/marketplace` — body (`StrategyCreate`):

```json
{ "name": "Momentum 20/50", "description": "MA crossover momentum", "signal_logic": "ema20 > ema50", "price_usd": 9.99, "is_free": false }
```

Errors: `201` on publish, `400` for an invalid listing, `404` for an unknown
strategy, `409` if already purchased/published.

## Modes & trading models — `backend/api/modes.py`, `backend/api/trading_models.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/modes` | — | Available trading modes and the active one |
| GET | `/api/v1/models` | 🔒 | List trading models |
| PUT | `/api/v1/models/preferences` | 🔒 | Set my model preference |

`GET /api/v1/modes` returns `{ "status": "ok", "active_mode": "...", "modes": [...] }`.

`PUT /api/v1/models/preferences` — body (`ModelPreferenceRequest`):

```json
{ "mode": "technical", "model_id": "ta-default" }
```

`mode` is a `Mode` enum from `trading_models.py`.

## Trading tools — `backend/api/trading_tools.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/trading-tools/sltp` | 🔒 | Create a stop-loss/take-profit order |
| GET | `/api/v1/trading-tools/sltp` | 🔒 | List my SL/TP orders |
| DELETE | `/api/v1/trading-tools/sltp/{order_id}` | 🔒 | Cancel an SL/TP order |
| GET | `/api/v1/trading-tools/multitimeframe/{ticker}` | — | Multi-timeframe signals |
| POST | `/api/v1/trading-tools/trade-plan` | 🔒 | Create a trade plan |
| GET | `/api/v1/trading-tools/trade-plan` | 🔒 | List my trade plans |
| DELETE | `/api/v1/trading-tools/trade-plan/{plan_id}` | 🔒 | Delete a trade plan |

`POST /api/v1/trading-tools/sltp` — body (`SLTPCreate`):

```json
{ "ticker": "AAPL", "entry_price": 228.0, "stop_loss": 215.0, "take_profit": 250.0, "quantity": 10, "notes": "Earnings play" }
```

`POST /api/v1/trading-tools/trade-plan` — body (`TradePlanCreate`):

```json
{ "ticker": "AAPL", "entry": 228.0, "target": 250.0, "stop": 215.0, "quantity": 10, "rationale": "Breakout retest", "timeframe": "1w" }
```

## Notifications — `backend/api/notify.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/notifications` | 🔒 | List notifications |
| POST | `/api/v1/notifications/{notif_id}/read` | 🔒 | Mark one as read |
| POST | `/api/v1/notifications/read-all` | 🔒 | Mark all as read |
| GET | `/api/v1/notifications/unread-count` | 🔒 | Unread count |
| POST | `/api/v1/notifications/send-test` | 🔒 | Send a test notification |

`POST /api/v1/notifications/send-test` — body (`TestNotifIn`): `{ "title": "Test", "body": "It works" }`.

## Engagement — `backend/api/engagement.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/engagement/streak` | 🔒 | My login streak |
| POST | `/api/v1/engagement/streak/claim-bonus` | 🔒 | Claim the daily streak bonus |
| GET | `/api/v1/engagement/achievements` | 🔒 | My achievements |
| POST | `/api/v1/engagement/achievements/check` | 🔒 | Re-evaluate achievements |
| POST | `/api/v1/engagement/backtests/progress` | 🔒 | Record backtest progress |
| GET | `/api/v1/engagement/daily-challenges` | 🔒 | Today's challenges |
| POST | `/api/v1/engagement/daily-challenges/{challenge_id}/claim` | 🔒 | Claim a challenge reward |
| POST | `/api/v1/engagement/daily-challenges/progress` | 🔒 | Record challenge progress (`requirement_type` query) |

Example — `GET /api/v1/engagement/streak` returns
`{ "status": "ok", "streak": { "current_streak": 3, "longest_streak": 9, "total_logins": 12 } }`-style data.

## Referral — `backend/api/referral.py`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/referral/generate` | 🔒 | Generate my referral code |
| GET | `/api/v1/referral/my-code` | 🔒 | My referral code |
| POST | `/api/v1/referral/use` | 🔒 | Redeem a referral code |
| GET | `/api/v1/referral/stats` | 🔒 | Referral stats |

`POST /api/v1/referral/use` — body (`UseCodeRequest`): `{ "code": "KING-AB12CD" }`.

Errors: `400` for an invalid/already-used code, `404` when no code exists yet.

## WebSocket — `backend/api/websocket.py`

| Protocol | Path | Description |
|----------|------|-------------|
| WS | `/ws/{client_id}` | Realtime price/signal stream (mounted outside `/api/v1`) |

Connect with a unique `client_id`; the server pushes JSON market updates for
the client's subscribed tickers.
