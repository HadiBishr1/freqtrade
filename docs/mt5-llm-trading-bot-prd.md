# PRD: Local Autonomous LLM-Guided MT5 BTC Trading Bot

## 1. Summary

Build a local-only autonomous BTC/crypto trading bot inside this Freqtrade repository. The bot will trade an MT5 demo account through the provided local `mt5rest-internal` REST API container, connect with `ConnectEx`, reuse the returned MT5 session `id` on every endpoint call, use an OpenAI-compatible LLM gateway for trade decisions, apply deterministic risk controls, persist all decisions/executions locally, and expose a minimal local frontend for monitoring agent performance.

This is not a public production product. It runs locally and does not need user authentication, user onboarding, public deployment, or multi-user flows.

## 2. Repository Fit

This repository is Freqtrade, a Python trading bot framework. Relevant existing components:

- CLI entrypoint: `freqtrade/main.py`
- Commands: `freqtrade/commands/*`
- Main worker loop pattern: `freqtrade/worker.py`
- Core bot orchestration: `freqtrade/freqtradebot.py`
- Strategy interface: `freqtrade/strategy/interface.py`
- Example strategy/template: `freqtrade/templates/sample_strategy.py`, `freqtrade/templates/base_strategy.py.j2`
- API server/frontend serving patterns: `freqtrade/rpc/api_server/*`
- Docs for strategies, REST API, UI, backtesting, hyperopt, and FreqAI under `docs/`

## 3. Does Freqtrade Have Built-In Strategies?

Freqtrade has the infrastructure for strategies, but it does not ship with production-ready profitable strategies that should simply be turned on.

It does include:

- `IStrategy`, the core strategy interface.
- `SampleStrategy`, which demonstrates indicators and signal generation.
- Strategy templates and `new-strategy` tooling.
- Indicator patterns using Pandas, TA-Lib, and `technical.qtpylib`.
- Strategy concepts such as stoploss, ROI, trailing stop, entry/exit signals, callbacks, and protections.
- FreqAI infrastructure, though this PRD assumes the primary intelligence comes from an external OpenAI-compatible LLM gateway.

For this project, we should reuse Freqtrade strategy concepts and indicator code patterns, but not treat `SampleStrategy` as a complete trading strategy.

## 4. Goals

### Primary Goals

1. Run locally on the user's machine.
2. Use the MT5 REST API container at `http://localhost:5015` by default.
3. Connect to MT5 with `GET /ConnectEx`.
4. Store the returned token and pass it as `id` on all MT5 API calls.
5. Trade BTC/crypto on a 10K demo account.
6. Use full autonomous LLM guidance through an OpenAI-compatible gateway.
7. Require no human approval during runtime.
8. Save all market snapshots, LLM decisions, risk checks, rejected decisions, and executions locally.
9. Provide a minimal local frontend to monitor performance.
10. Reuse useful Freqtrade patterns and utilities so the feature is not built 100% from scratch.

### Secondary Goals

1. Support dry-run mode.
2. Support configurable model, LLM base URL, and API key.
3. Support configurable risk limits.
4. Support BTC symbol auto-discovery.
5. Allow future deeper integration with normal Freqtrade abstractions.

## 5. Non-Goals

1. No public SaaS deployment.
2. No user authentication.
3. No multi-user account management.
4. No real-money trading support in MVP.
5. No promise of profitability.
6. No need to reuse the full existing Freqtrade UI bundle.
7. No need to support all MT5 symbols in MVP.
8. No need to support multiple MT5 accounts in MVP.

## 6. Recommended Implementation Approach

### 6.1 Dedicated MT5 LLM Module With Freqtrade Reuse

Do not force MT5 REST into Freqtrade's CCXT exchange layer for MVP. MT5's session/account/order model is different enough that this would slow the first working version.

Recommended module:

```text
freqtrade/mt5_llm/
  __init__.py
  api.py
  client.py
  config.py
  engine.py
  frontend.py
  indicators.py
  llm.py
  models.py
  persistence.py
  prompts.py
  risk.py
  schema.py
  strategy.py
```

Recommended command:

```bash
freqtrade mt5-llm-trade --config user_data/config_mt5_llm.json
```

Optional web-only command:

```bash
freqtrade mt5-llm-web --config user_data/config_mt5_llm.json
```

Preferred MVP runtime:

```bash
freqtrade mt5-llm-trade --config user_data/config_mt5_llm.json --dashboard-port 8088
```

### 6.2 What To Reuse From Freqtrade

| Freqtrade Area | Reuse Plan |
|---|---|
| CLI command patterns | Add a new command under `freqtrade/commands`. |
| Worker loop pattern | Reuse the idea of a controlled infinite loop with graceful shutdown. |
| Strategy concepts | Mirror `populate_indicators`, entry/exit signals, stoploss, ROI, and callbacks. |
| Indicator examples | Reuse ideas from `SampleStrategy`: RSI, MACD, Bollinger Bands, TEMA, SAR, volume checks. |
| Technical libraries | Use Pandas, TA-Lib, and `technical.qtpylib` where available. |
| Logging style | Use existing Freqtrade logging conventions. |
| FastAPI server patterns | Reuse `freqtrade/rpc/api_server` structure ideas for local dashboard endpoints. |
| Local persistence idea | Use SQLite locally, but with MT5 LLM-specific tables. |
| Performance concepts | Track PnL, equity, drawdown, open trades, win/loss, and realized/unrealized PnL. |

### 6.3 What Not To Reuse For MVP

Avoid making these hard dependencies initially:

- CCXT exchange abstraction.
- Standard `FreqtradeBot` order execution path.
- Existing Freqtrade trade tables, unless mapped carefully later.
- Full existing Freqtrade UI bundle.

## 7. MT5 REST Runtime

The local MT5 REST API should be started with:

```bash
docker pull ocapitalglobal/mt5rest-internal:latest

docker run -d --name mt5rest-internal -p 5015:80 \
  -e Auth=false -e InternalBuild=true \
  ocapitalglobal/mt5rest-internal:latest
```

Default base URL:

```text
http://localhost:5015
```

Health check:

```http
GET /Ping
```

Expected response:

```text
OK
```

## 8. MT5 Connection Flow

Use `ConnectEx`:

```http
GET /ConnectEx?user={user}&password={password}&server={server}
```

On success, it returns a plain-text token. The bot must store this token and pass it as `id` to all later MT5 calls.

Important connection/status endpoints:

| Purpose | Endpoint |
|---|---|
| Connect | `GET /ConnectEx` |
| Check connection | `GET /ConnectionStatus?id={id}` |
| Reconnect/check | `GET /CheckConnect?id={id}` |
| Disconnect | `GET /Disconnect?id={id}` |

## 9. MT5 API Endpoints Used

### Account and Orders

| Purpose | Endpoint |
|---|---|
| Account summary | `GET /AccountSummary?id={id}` |
| Account details | `GET /AccountDetails?id={id}` |
| Open orders | `GET /OpenedOrders?id={id}` |
| Order history | `GET /OrderHistory` or `GET /OrderHistoryPagination` |

### Symbols and Market Data

| Purpose | Endpoint |
|---|---|
| Symbol list | `GET /SymbolList?id={id}` |
| Symbol params | `GET /SymbolParams?id={id}&symbol={symbol}` |
| Trade session status | `GET /IsTradeSession?id={id}&symbol={symbol}` |
| Latest quote | `GET /GetQuote?id={id}&symbol={symbol}` |
| Recent bars | `GET /PriceHistoryEx?id={id}&symbol={symbol}&from={now}&numBars={n}&timeFrame={minutes}` |

### Trading

Use safe endpoints in MVP:

| Purpose | Endpoint |
|---|---|
| Open order | `GET /OrderSendSafe` |
| Modify SL/TP | `GET /OrderModifySafe` |
| Close order | `GET /OrderCloseSafe` |

## 10. BTC Symbol Resolution

The bot must support explicit and automatic BTC symbol selection.

If configured:

```json
{
  "mt5": {
    "symbol": "BTCUSD"
  }
}
```

Validate that:

1. The symbol exists in `SymbolList`.
2. `SymbolParams` returns valid details.
3. `IsTradeSession` is true before trading.

If not configured, auto-detect symbols containing:

- `BTC`
- `XBT`
- `BITCOIN`

Prefer common symbols:

1. `BTCUSD`
2. `BTCUSD.`
3. `BTCUSDm`
4. `BTCUSD.r`
5. `BTCUSDT`
6. `BTC`

If no active BTC symbol is found, the bot should wait and retry.

## 11. Strategy Design

### 11.1 Hybrid Strategy

The trading engine should use a hybrid approach:

1. Deterministic analysis layer computes indicators and candidate signals.
2. LLM policy layer decides what to do from structured context.
3. Deterministic risk layer validates or rejects the LLM decision.
4. Execution layer sends validated orders to MT5.

The LLM is autonomous, but it does not bypass risk controls.

### 11.2 Indicators To Use

Use a compact indicator set inspired by Freqtrade's sample strategy:

| Indicator | Purpose |
|---|---|
| RSI | Momentum and overbought/oversold context. |
| EMA 9/21/50 or TEMA | Short/medium trend direction. |
| MACD | Momentum confirmation. |
| Bollinger Bands width/percent | Volatility and price location. |
| ATR | Volatility-based stop sizing. |
| Recent returns | Short-term momentum. |
| Spread percentage | Liquidity/trade quality filter. |
| Volume/tick volume trend | Activity filter. |

### 11.3 Strategy Modes

MVP should support these modes:

#### Mode A: LLM-only with deterministic context

The LLM receives structured market/account context and decides the action. Risk controls still validate all decisions.

#### Mode B: Signal-gated LLM

The deterministic strategy must first produce a candidate trade signal before the LLM can open a new position. The LLM can still hold, close, or modify positions.

Recommended MVP default: **Signal-gated LLM**.

#### Mode C: Deterministic-only dry-run baseline

No LLM execution. The deterministic signal engine logs what it would do. Useful for debugging indicators, symbol params, and dashboard metrics.

### 11.4 Candidate Signal Example

```json
{
  "candidate_bias": "LONG",
  "trend_state": "UPTREND",
  "momentum_state": "POSITIVE",
  "volatility_state": "NORMAL",
  "spread_state": "ACCEPTABLE",
  "suggested_stoploss": 62000.0,
  "suggested_takeprofit": 65000.0,
  "suggested_volume": 0.01
}
```

## 12. LLM Gateway

The LLM gateway must be OpenAI-compatible.

Config:

```json
{
  "llm": {
    "base_url": "https://your-llm-gateway.example.com/v1",
    "api_key": "YOUR_KEY",
    "model": "your-model-name",
    "temperature": 0.1,
    "timeout_seconds": 60
  }
}
```

Call format:

```http
POST {base_url}/chat/completions
Authorization: Bearer {api_key}
Content-Type: application/json
```

The bot should request JSON-only responses.

## 13. LLM Decision Contract

The LLM must return strict JSON only.

Allowed actions:

- `HOLD`
- `OPEN_LONG`
- `OPEN_SHORT`
- `CLOSE_POSITION`
- `MODIFY_POSITION`

Decision schema:

```json
{
  "action": "HOLD",
  "symbol": "BTCUSD",
  "confidence": 0.72,
  "reason": "Momentum is positive but spread is elevated.",
  "order": {
    "side": "BUY",
    "volume": 0.01,
    "stoploss": 62000.0,
    "takeprofit": 65000.0
  },
  "position_management": {
    "ticket": 123456789,
    "modify_stoploss": null,
    "modify_takeprofit": null,
    "close_volume": null
  },
  "risk": {
    "max_loss_usd": 50.0,
    "invalid_if_price_above": null,
    "invalid_if_price_below": null
  }
}
```

Invalid JSON or unsupported actions must become `HOLD`.

## 14. Risk Guardrails

Even though trading is autonomous, hard deterministic guardrails are required.

Default risk config:

```json
{
  "risk": {
    "demo_only": true,
    "max_open_positions": 1,
    "max_position_notional_pct": 20,
    "max_risk_per_trade_pct": 1,
    "max_daily_loss_pct": 3,
    "max_total_drawdown_pct": 10,
    "min_confidence_to_trade": 0.65,
    "require_stoploss": true,
    "require_takeprofit": false,
    "max_spread_pct": 0.25,
    "cooldown_seconds_after_trade": 300,
    "loop_interval_seconds": 60,
    "allow_short": false
  }
}
```

Reject or downgrade to `HOLD` when:

- Account is not demo while `demo_only=true`.
- Account summary has `synced=false`.
- Trading session is closed.
- Symbol is invalid.
- Quote is stale.
- Spread exceeds max spread.
- LLM confidence is too low.
- Stoploss is missing when required.
- Stoploss risk exceeds max risk per trade.
- Position count exceeds max open positions.
- Daily loss exceeds max daily loss.
- Total drawdown exceeds max total drawdown.
- Required margin is too high.
- LLM output is invalid.

## 15. Trading Loop

Startup:

1. Load config.
2. Check `/Ping`.
3. Call `/ConnectEx`.
4. Store returned `id`.
5. Fetch account summary/details.
6. Verify demo account if required.
7. Resolve BTC symbol.
8. Fetch symbol params.
9. Start dashboard API if enabled.
10. Start trading loop.

Each loop:

1. Check connection.
2. Fetch account summary.
3. Fetch open orders.
4. Fetch latest quote.
5. Fetch recent candles across configured timeframes.
6. Compute indicators and candidate signals.
7. Build LLM prompt.
8. Call LLM gateway.
9. Parse and validate decision JSON.
10. Run risk checks.
11. Execute, modify, close, or hold.
12. Persist snapshot, decision, risk result, and execution.
13. Sleep until next interval.

## 16. Execution Mapping

### Open Long

```http
GET /OrderSendSafe?id={id}&symbol={symbol}&operation=Buy&volume={volume}&price=0&slippage={slippage}&stoploss={stoploss}&takeprofit={takeprofit}&comment=llm-btc-bot&placedType=ByExpert
```

### Open Short

```http
operation=Sell
```

Only allowed when `risk.allow_short=true` and the symbol/account permits shorting.

### Modify Position

```http
GET /OrderModifySafe?id={id}&ticket={ticket}&stoploss={stoploss}&takeprofit={takeprofit}
```

### Close Position

```http
GET /OrderCloseSafe?id={id}&ticket={ticket}&lots={lots_or_0}&price=0&slippage={slippage}&comment=llm-btc-bot-close
```

## 17. Minimal Frontend

### 17.1 Goal

Provide a minimal local dashboard to track how the autonomous agent is performing.

The frontend does not need authentication. It should only bind locally by default.

Default URL:

```text
http://localhost:8088
```

### 17.2 Recommended Implementation

Use FastAPI for local API endpoints and serve a small static HTML/JS page, following the pattern used by Freqtrade's existing API server.

Recommended files:

```text
freqtrade/mt5_llm/api.py
freqtrade/mt5_llm/frontend.py
freqtrade/mt5_llm/static/index.html
freqtrade/mt5_llm/static/app.js
freqtrade/mt5_llm/static/styles.css
```

The frontend can be plain HTML + JavaScript + Chart.js or lightweight React. For MVP, plain HTML/JS is enough.

### 17.3 Dashboard Views

The dashboard should show:

1. Bot status: running, paused, stopped, error.
2. MT5 connection status.
3. Account balance/equity/free margin/margin level.
4. Current BTC symbol, bid, ask, spread, last quote time.
5. Open positions with ticket, side, volume, open price, SL, TP, unrealized PnL.
6. Recent closed trades and realized PnL.
7. Latest LLM decision and reason.
8. Latest risk rejection reason, if any.
9. Equity curve.
10. Daily PnL.
11. Win/loss count.
12. Drawdown.
13. Recent execution log.

### 17.4 Local API Endpoints

Recommended local dashboard endpoints:

| Endpoint | Purpose |
|---|---|
| `GET /api/mt5-llm/status` | Bot, MT5, LLM, and run status. |
| `GET /api/mt5-llm/account` | Latest account summary. |
| `GET /api/mt5-llm/positions` | Current open positions. |
| `GET /api/mt5-llm/performance` | PnL, drawdown, win/loss, equity curve. |
| `GET /api/mt5-llm/decisions?limit=50` | Recent LLM decisions. |
| `GET /api/mt5-llm/executions?limit=50` | Recent executions. |
| `GET /api/mt5-llm/snapshots?limit=100` | Recent market snapshots. |
| `GET /api/mt5-llm/config` | Sanitized runtime config. |

## 18. Persistence

Use SQLite for MVP.

Recommended tables:

### `runs`

- `id`
- `started_at`
- `ended_at`
- `status`
- `config_hash`
- `mt5_account`
- `symbol`

### `market_snapshots`

- `id`
- `run_id`
- `created_at`
- `symbol`
- `bid`
- `ask`
- `spread`
- `account_balance`
- `account_equity`
- `account_margin`
- `account_free_margin`
- `open_positions_json`
- `indicators_json`
- `bars_json`

### `llm_decisions`

- `id`
- `run_id`
- `snapshot_id`
- `created_at`
- `prompt_hash`
- `raw_response`
- `parsed_decision_json`
- `valid`
- `rejection_reason`

### `risk_checks`

- `id`
- `run_id`
- `decision_id`
- `created_at`
- `approved`
- `reasons_json`
- `normalized_order_json`

### `executions`

- `id`
- `run_id`
- `decision_id`
- `created_at`
- `action`
- `request_json`
- `response_json`
- `success`
- `ticket`
- `error`

### `performance_points`

- `id`
- `run_id`
- `created_at`
- `balance`
- `equity`
- `realized_pnl`
- `unrealized_pnl`
- `drawdown_pct`

## 19. Configuration Example

```json
{
  "mt5": {
    "base_url": "http://localhost:5015",
    "user": 123456789,
    "password": "your-mt5-password",
    "server": "YourBroker-Demo",
    "connect_timeout_seconds": 60,
    "symbol": "BTCUSD",
    "slippage": 0
  },
  "llm": {
    "base_url": "https://your-llm-gateway.example.com/v1",
    "api_key": "your-llm-gateway-key",
    "model": "your-model",
    "temperature": 0.1,
    "timeout_seconds": 60
  },
  "strategy": {
    "mode": "signal_gated_llm",
    "primary_timeframe_minutes": 1,
    "secondary_timeframes_minutes": [5, 15, 60],
    "bars_per_timeframe": 100,
    "quote_max_age_seconds": 30
  },
  "risk": {
    "demo_only": true,
    "max_open_positions": 1,
    "max_position_notional_pct": 20,
    "max_risk_per_trade_pct": 1,
    "max_daily_loss_pct": 3,
    "max_total_drawdown_pct": 10,
    "min_confidence_to_trade": 0.65,
    "require_stoploss": true,
    "require_takeprofit": false,
    "max_spread_pct": 0.25,
    "cooldown_seconds_after_trade": 300,
    "loop_interval_seconds": 60,
    "allow_short": false
  },
  "dashboard": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 8088
  },
  "storage": {
    "sqlite_path": "user_data/mt5_llm_bot.sqlite",
    "log_path": "user_data/logs/mt5_llm_bot.log"
  }
}
```

## 20. Acceptance Criteria

MVP is complete when:

1. The MT5 REST container can be run locally.
2. The bot can call `/Ping` successfully.
3. The bot can connect via `/ConnectEx`.
4. The bot stores and reuses the returned `id`.
5. The bot can fetch account summary and verify demo mode.
6. The bot can resolve or validate a BTC symbol.
7. The bot can fetch BTC quote and candle history.
8. The bot computes deterministic indicators.
9. The bot can call the OpenAI-compatible LLM gateway.
10. The bot parses strict JSON LLM decisions.
11. The bot applies deterministic risk checks.
12. The bot can autonomously hold/open/modify/close positions.
13. The bot can place a BTC demo trade with `OrderSendSafe`.
14. The bot can close a BTC demo trade with `OrderCloseSafe`.
15. The bot logs and persists all decisions and executions.
16. The dashboard shows status, account, open positions, decisions, executions, PnL, and equity curve.
17. The bot continues running until manually stopped.
18. The bot refuses to trade when guardrails fail.

## 21. Milestones

### Milestone 1: MT5 Client

- Implement HTTP client.
- Implement `/Ping`, `/ConnectEx`, `/ConnectionStatus`.
- Implement account, symbol, quote, candles, open orders.
- Implement safe order send/modify/close wrappers.

### Milestone 2: Strategy and Risk Engine

- Implement candle normalization.
- Implement indicators.
- Implement candidate signal generation.
- Implement risk guardrails.
- Implement dry-run mode.

### Milestone 3: LLM Integration

- Implement OpenAI-compatible client.
- Implement prompt builder.
- Implement JSON schema validation.
- Implement fallback to `HOLD` on invalid output.

### Milestone 4: Persistence

- Implement SQLite schema.
- Store snapshots, decisions, risk checks, executions, and performance points.

### Milestone 5: Minimal Frontend

- Implement FastAPI dashboard endpoints.
- Implement local static dashboard.
- Add equity/PnL chart and decision/execution tables.

### Milestone 6: End-to-End Demo

- Run local MT5 REST container.
- Connect to demo account.
- Resolve BTC symbol.
- Run bot in dry-run.
- Run bot in demo execution mode with tiny size.
- Open, modify, and close a BTC demo trade.
- Validate dashboard and persistence.

## 22. Final Recommendation

Build the MT5 LLM bot as a dedicated module and command inside Freqtrade, not as an external standalone project. Reuse Freqtrade's CLI patterns, strategy concepts, indicator conventions, logging style, API server patterns, and performance metric ideas. Do not force the MVP into the CCXT/Freqtrade exchange execution path yet. This gives the fastest path to a working local autonomous MT5 BTC bot while preserving a clean path for deeper Freqtrade integration later.
