# MT5 LLM Trading Bot - Product Requirements Document

## Executive Summary

Build a fully autonomous Bitcoin trading bot that uses MT5 REST API for execution and an OpenAI-compatible LLM gateway for trade decisions. The bot runs locally on Docker with a web dashboard for monitoring performance. This is a hackathon project focused on getting a working prototype, not production-grade security.

**Architecture Decision**: Create a new standalone module `freqtrade/mt5_llm/` within the existing Freqtrade repo. Borrow proven patterns (worker loop, strategy interface, persistence, FastAPI) but avoid tightly coupling to Freqtrade's CCXT-based exchange layer since MT5 uses a different API paradigm.

---

## System Architecture

```
Docker Compose Stack:
┌─────────────────────────────────────────────────────────────┐
│  MT5 REST API (5015) ◄──► Trading Bot (Python) ◄──► Dashboard (8088)  │
│                                    │                                  │
│                              LLM Gateway (OpenAI API)                │
└─────────────────────────────────────────────────────────────┘
```

**Components:**
- **MT5 REST API Container**: `ocapitalglobal/mt5rest-internal:latest` on port 5015 (no auth)
- **Trading Bot**: Python module with worker loop, MT5 client, LLM integration, risk management, SQLite persistence
- **Dashboard**: FastAPI + vanilla HTML/JS with real-time WebSocket updates
- **LLM Gateway**: OpenAI-compatible API endpoint for trade decision reasoning

---

## MT5 Integration

### Connection Flow
```python
token = mt5_client.connect_ex(user, password, server)
# All subsequent calls pass token as 'id' parameter
account_info = mt5_client.account_summary(id=token)
order_result = mt5_client.order_send_safe(id=token, ...)
```

### Required Endpoints
- **Connection**: `GET /ConnectEx`, `/CheckConnect`, `/Disconnect`
- **Account**: `/AccountSummary`, `/OpenedOrders`, `/ClosedOrders`
- **Trading**: `/OrderSendSafe`, `/OrderModifySafe`, `/OrderCloseSafe`, `/OrderCancelTask`
- **Market Data**: `/GetQuote`, `/PriceHistoryV2`, `/SymbolParams`
- **Subscriptions**: `/Subscribe`, `/OnQuote` (optional WebSocket streaming)

### MT5 Client (`freqtrade/mt5_llm/mt5_client.py`)
```python
class MT5Client:
    def __init__(self, base_url: str = "http://localhost:5015"):
        self.base_url = base_url
        self.token = None
        self.session = requests.Session()
    
    def connect_ex(self, user: int, password: str, server: str) -> str
    def account_summary(self) -> dict
    def get_opened_orders(self) -> list
    def get_price_history(self, symbol: str, timeframe: str, from_dt: str, to_dt: str) -> list
    def order_send_safe(self, symbol: str, operation: str, volume: float, 
                       price: float = 0, stoploss: float = 0, takeprofit: float = 0) -> dict
    def order_close_safe(self, ticket: int, lots: float = 0) -> dict
```

**Key Considerations:**
- All requests are GET (unusual but per swagger spec)
- Token must be passed as `id` query parameter on every call
- Handle connection drops gracefully (auto-reconnect)
- Error responses return HTTP 201 with `ExceptionResult` JSON

---

## Trading Strategy & LLM Integration

### Hybrid Approach (Recommended for MVP)
1. **Deterministic Indicators** - Calculate RSI, EMA, MACD, Bollinger Bands, ATR
2. **LLM Policy Layer** - LLM reasons over indicators + market context → outputs decision
3. **Risk Guardrails** - Hard-coded rules that override LLM if violated

### Indicator Calculation (`freqtrade/mt5_llm/indicators.py`)
```python
def calculate_indicators(df: pd.DataFrame) -> pd.DataFrame:
    # Input: OHLCV dataframe [time, open, high, low, close, volume]
    # Output: Same dataframe with indicator columns
    df['rsi_14'] = ta.momentum.RSIIndicator(df['close'], window=14).rsi()
    df['ema_20'] = ta.trend.EMAIndicator(df['close'], window=20).ema_indicator()
    df['ema_50'] = ta.trend.EMAIndicator(df['close'], window=50).ema_indicator()
    df['macd'] = ta.trend.MACD(df['close']).macd()
    df['macd_signal'] = ta.trend.MACD(df['close']).macd_signal()
    df['bb_upper'] = ta.volatility.BollingerBands(df['close']).bollinger_hband()
    df['bb_lower'] = ta.volatility.BollingerBands(df['close']).bollinger_lband()
    df['atr_14'] = ta.volatility.AverageTrueRange(df['high'], df['low'], df['close']).average_true_range()
    return df
```

### LLM Decision Interface (`freqtrade/mt5_llm/llm.py`)
```python
class LLMTrader:
    def __init__(self, api_base: str, api_key: str, model: str):
        self.client = OpenAI(base_url=api_base, api_key=api_key)
        self.model = model
    
    def decide(self, context: TradeContext) -> TradeDecision:
        # Input: indicators, current position, account state, recent candles
        # Output: {action, confidence, reasoning, parameters}
        prompt = self._build_prompt(context)
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"}
        )
        return self._parse_response(response)
```

**System Prompt:**
```
You are an autonomous Bitcoin trading bot. Analyze the market context and decide on an action.

Rules:
- Trade only BTC/USD or BTC/USDC
- Max 1 open position at a time
- Always set stoploss (1-2% risk per trade)
- Use technical indicators to inform decisions
- Output JSON with: action, confidence, reasoning, parameters

Actions:
- HOLD: No trade, wait for better setup
- OPEN_LONG: Buy BTC (bullish signal)
- OPEN_SHORT: Sell BTC (bearish signal)
- CLOSE_POSITION: Exit current position
- MODIFY_POSITION: Adjust SL/TP of open position
```

### Risk Management (`freqtrade/mt5_llm/risk.py`)
```python
class RiskManager:
    def __init__(self, config: RiskConfig):
        self.max_open_positions = 1
        self.max_risk_per_trade = 0.01  # 1% of equity
        self.max_daily_loss = 0.03      # 3% of starting equity
        self.max_drawdown = 0.10        # 10% from peak
        self.min_confidence = 0.65      # LLM confidence threshold
        self.require_stoploss = True
        self.max_spread = 0.0025        # 0.25% max spread
        self.cooldown_seconds = 300     # 5 min between trades
    
    def validate_decision(self, decision: TradeDecision, 
                         account: AccountSummary, 
                         market: MarketSnapshot) -> ValidationResult:
        # Check all risk rules, calculate position size based on risk %
        pass
```

---

## Worker Loop & State Machine (`freqtrade/mt5_llm/engine.py`)

```python
class TradingEngine:
    def __init__(self, config: BotConfig):
        self.mt5 = MT5Client(config.mt5_url)
        self.llm = LLMTrader(config.llm_api, config.llm_key, config.llm_model)
        self.risk = RiskManager(config.risk)
        self.db = Database(config.db_url)
        self.state = BotState.STOPPED
    
    def run(self):
        """Main worker loop"""
        self.mt5.connect_ex(...)
        self.state = BotState.RUNNING
        
        while self.state == BotState.RUNNING:
            try:
                self._iteration()
                time.sleep(self.config.throttle_seconds)
            except KeyboardInterrupt:
                break
            except Exception as e:
                logger.error(f"Iteration failed: {e}")
                time.sleep(10)
        
        self._shutdown()
    
    def _iteration(self):
        """Single iteration of the trading loop"""
        # 1. Fetch current market data
        candles = self.mt5.get_price_history("BTCUSD", "H1", ...)
        df = calculate_indicators(candles)
        
        # 2. Get account state
        account = self.mt5.account_summary()
        open_orders = self.mt5.get_opened_orders()
        
        # 3. Build context for LLM
        context = TradeContext(
            indicators=df.tail(1).to_dict(),
            account=account,
            open_positions=open_orders,
            recent_candles=df.tail(20).to_dict()
        )
        
        # 4. Get LLM decision
        decision = self.llm.decide(context)
        
        # 5. Validate with risk manager
        validation = self.risk.validate_decision(decision, account, context)
        if not validation.is_valid:
            logger.info(f"Risk check failed: {validation.reason}")
            self.db.log_risk_check(decision, validation)
            return
        
        # 6. Execute decision
        self._execute_decision(decision)
        
        # 7. Persist state
        self.db.log_iteration(context, decision, validation)
```

---

## Persistence Layer (`freqtrade/mt5_llm/persistence.py`)

Use SQLAlchemy + SQLite (same pattern as Freqtrade):

```python
class Database:
    def __init__(self, db_url: str = "sqlite:///mt5_bot.db"):
        self.engine = create_engine(db_url)
        self.Session = sessionmaker(bind=self.engine)
        Base.metadata.create_all(self.engine)
    
    def log_iteration(self, context, decision, validation)
    def log_trade(self, order: dict, action: str)
    def get_performance_history(self, days: int = 30) -> list
    def get_trade_history(self, limit: int = 100) -> list
```

**Tables:**
- `runs`: Bot session tracking
- `iterations`: Each loop iteration with context/decision
- `trades`: Open/closed positions
- `risk_checks`: Risk validation results
- `performance_points`: Equity curve over time

---

## Dashboard & Monitoring (`freqtrade/mt5_llm/dashboard.py`)

```python
from fastapi import FastAPI, WebSocket
from fastapi.staticfiles import StaticFiles

app = FastAPI()

# REST endpoints
@app.get("/api/status")
def status():
    """Bot state, connection status, current iteration"""

@app.get("/api/account")
def account():
    """Balance, equity, margin, profit"""

@app.get("/api/positions")
def positions():
    """Current open positions"""

@app.get("/api/trades")
def trades(limit: int = 100):
    """Historical trades"""

@app.get("/api/performance")
def performance(days: int = 30):
    """Equity curve data points"""

@app.get("/api/decisions")
def decisions(limit: int = 50):
    """Recent LLM decisions with reasoning"""

# WebSocket for real-time updates
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """Stream live updates to frontend"""

# Serve static HTML/JS
app.mount("/", StaticFiles(directory="freqtrade/mt5_llm/static", html=True))
```

**Frontend** (`freqtrade/mt5_llm/static/index.html`):
- Single-page app with vanilla JS (no React/Vue for simplicity)
- Fetch data from REST endpoints every 5 seconds
- Display: Account summary, current position, recent trades table, equity curve chart (Chart.js), LLM decision log with reasoning

---

## Docker Compose Setup

### `docker-compose.mt5.yml`
```yaml
version: '3.8'

services:
  mt5rest:
    image: ocapitalglobal/mt5rest-internal:latest
    container_name: mt5rest
    ports:
      - "5015:80"
    environment:
      - Auth=false
      - InternalBuild=true
    restart: unless-stopped

  trading-bot:
    build:
      context: .
      dockerfile: Dockerfile.mt5
    container_name: mt5-trading-bot
    depends_on:
      - mt5rest
    environment:
      - MT5_URL=http://mt5rest:80
      - MT5_USER=${MT5_USER}
      - MT5_PASSWORD=${MT5_PASSWORD}
      - MT5_SERVER=${MT5_SERVER}
      - LLM_API_BASE=${LLM_API_BASE}
      - LLM_API_KEY=${LLM_API_KEY}
      - LLM_MODEL=${LLM_MODEL}
    volumes:
      - ./user_data/mt5_llm:/app/user_data/mt5_llm
    restart: unless-stopped

  dashboard:
    build:
      context: .
      dockerfile: Dockerfile.mt5
    container_name: mt5-dashboard
    command: python -m freqtrade.mt5_llm.dashboard
    ports:
      - "8088:8088"
    depends_on:
      - trading-bot
    environment:
      - DB_URL=sqlite:////app/user_data/mt5_llm/mt5_bot.db
    volumes:
      - ./user_data/mt5_llm:/app/user_data/mt5_llm
    restart: unless-stopped
```

### `Dockerfile.mt5`
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements-mt5.txt .
RUN pip install --no-cache-dir -r requirements-mt5.txt

# Copy bot code
COPY freqtrade/mt5_llm/ ./freqtrade/mt5_llm/

# Create data directory
RUN mkdir -p /app/user_data/mt5_llm

CMD ["python", "-m", "freqtrade.mt5_llm.engine"]
```

### `requirements-mt5.txt`
```
requests>=2.31.0
pandas>=2.0.0
ta>=0.10.2
openai>=1.0.0
sqlalchemy>=2.0.0
fastapi>=0.104.0
uvicorn>=0.24.0
websockets>=12.0
python-dotenv>=1.0.0
```

---

## Configuration (`user_data/mt5_llm/config.json`)

```json
{
  "mt5": {
    "url": "http://localhost:5015",
    "user": 12345678,
    "password": "your_password",
    "server": "MetaQuotes-Demo"
  },
  "llm": {
    "api_base": "https://your-llm-gateway.com/v1",
    "api_key": "your_api_key",
    "model": "gpt-4"
  },
  "trading": {
    "symbol": "BTCUSD",
    "timeframe": "H1",
    "throttle_seconds": 60,
    "max_open_positions": 1,
    "risk_per_trade": 0.01
  },
  "risk": {
    "max_daily_loss": 0.03,
    "max_drawdown": 0.10,
    "min_confidence": 0.65,
    "require_stoploss": true,
    "max_spread": 0.0025,
    "cooldown_seconds": 300
  },
  "dashboard": {
    "port": 8088,
    "update_interval": 5
  },
  "database": {
    "url": "sqlite:///user_data/mt5_llm/mt5_bot.db"
  }
}
```

---

## File Structure

```
freqtrade/mt5_llm/
├── __init__.py
├── config.py              # Configuration models (pydantic)
├── mt5_client.py          # MT5 REST API wrapper
├── indicators.py          # Technical indicator calculations
├── llm.py                 # LLM integration
├── risk.py                # Risk management
├── engine.py              # Main trading engine/worker loop
├── persistence.py         # SQLAlchemy models and database
├── dashboard.py           # FastAPI dashboard server
├── models.py              # Pydantic models for API responses
├── prompts.py             # LLM prompt templates
├── static/
│   ├── index.html         # Dashboard frontend
│   ├── style.css
│   └── app.js
└── tests/
    ├── test_mt5_client.py
    ├── test_indicators.py
    ├── test_llm.py
    └── test_risk.py

user_data/mt5_llm/
├── config.json            # Bot configuration
├── .env                   # Secrets (MT5 credentials, LLM API key)
└── mt5_bot.db            # SQLite database (created at runtime)

docker-compose.mt5.yml     # Docker Compose for full stack
Dockerfile.mt5             # Dockerfile for bot and dashboard
requirements-mt5.txt       # Python dependencies
README.mt5.md              # Setup and usage guide
```

---

## Implementation Plan

### Phase 1: MT5 Client & Basic Trading (2-3 days)
- Create `freqtrade/mt5_llm/` module structure
- Implement `MT5Client` with connection management
- Test connection with demo account
- Implement basic order placement (market buy/sell)
- Implement position closing
- Fetch OHLCV data and convert to pandas DataFrame

### Phase 2: Indicators & LLM Integration (2-3 days)
- Calculate technical indicators (RSI, EMA, MACD, BB, ATR)
- Implement `LLMTrader` with OpenAI-compatible client
- Design system prompt and context formatting
- Parse LLM JSON responses
- Test LLM decisions with mock data

### Phase 3: Risk Management & Engine (1-2 days)
- Implement `RiskManager` with all validation rules
- Build `TradingEngine` worker loop
- Wire together MT5, indicators, LLM, and risk
- Add error handling and reconnection logic
- Test with dry-run mode (log decisions, no execution)

### Phase 4: Persistence & Logging (1 day)
- Define SQLAlchemy models for all tables
- Implement `Database` class with logging methods
- Add iteration/trade/risk logging to engine
- Query methods for dashboard data

### Phase 5: Dashboard (1-2 days)
- Create FastAPI app with REST endpoints
- Implement WebSocket for real-time updates
- Build HTML/JS frontend with charts
- Display account, positions, trades, equity curve, decisions
- Test dashboard with live bot data

### Phase 6: Docker & Deployment (1 day)
- Write `Dockerfile.mt5`
- Create `docker-compose.mt5.yml`
- Test full stack locally
- Document setup and running instructions

**Total Estimated Time: 8-12 days** (can be compressed with parallel work)

---

## Key Design Decisions

### 1. Why Not Use Freqtrade's Exchange Layer?
- MT5 uses a completely different REST API (not CCXT-compatible)
- MT5 returns data in different formats (order structure, candle data)
- Building a CCXT adapter for MT5 would be more work than a direct client
- For a hackathon, direct integration is faster and simpler

**Decision**: Create standalone MT5 client, borrow worker loop and strategy patterns from Freqtrade but don't inherit from `Exchange` class.

### 2. Why Not Use FreqAI?
- FreqAI is designed for traditional ML models (XGBoost, LightGBM, PyTorch)
- LLM integration is different: no training phase, text-based reasoning, no feature engineering pipeline needed
- Simpler to call directly from strategy

**Decision**: Integrate LLM directly in the trading engine, not through FreqAI.

### 3. Why Hybrid Strategy (Indicators + LLM)?
- Pure LLM trading has risks: hallucinations, high latency, cost per decision, no guaranteed consistency
- Hybrid approach provides: deterministic risk guardrails, technical indicators as structured context, LLM reasons over indicators, risk manager can override LLM

**Decision**: Calculate indicators → feed to LLM → validate with risk manager → execute

### 4. Why SQLite for Persistence?
- Zero configuration, file-based, sufficient for single-bot scenario
- Same pattern as Freqtrade (proven)
- Hackathon scope doesn't need distributed DB

**Decision**: SQLite with SQLAlchemy ORM

### 5. Why Vanilla JS Frontend?
- No build step, fast to develop and iterate, sufficient for simple dashboard
- Reduces dependencies and complexity

**Decision**: HTML + vanilla JS + Chart.js

---

## Success Criteria

### Minimum Viable Product (MVP)
- Bot connects to MT5 demo account
- Fetches BTC price history and calculates indicators
- LLM makes trade decisions based on indicators
- Risk manager validates decisions
- Bot executes trades (open/close positions)
- Persists all activity to SQLite
- Dashboard shows account balance, positions, trades, equity curve
- Runs continuously on Docker for 24+ hours without crashes

### Stretch Goals (if time permits)
- Add backtesting mode (replay historical data)
- Telegram notifications for trades
- Multiple timeframes (H1 for signals, D1 for trend)
- More sophisticated LLM prompts (few-shot examples)
- Position sizing based on Kelly criterion or volatility
- Webhook integration (entry/exit notifications)

---

## Getting Started

### Prerequisites
- Docker & Docker Compose
- MT5 demo account credentials (user, password, server)
- OpenAI-compatible LLM API endpoint and key
- Python 3.11+ (for local development)

### Quick Start

```bash
# 1. Clone repo and navigate to it
cd /Users/hadibishr/oxyz/freqtrade

# 2. Create .env file with credentials
cat > user_data/mt5_llm/.env << EOF
MT5_USER=12345678
MT5_PASSWORD=your_password
MT5_SERVER=MetaQuotes-Demo
LLM_API_BASE=https://your-gateway.com/v1
LLM_API_KEY=your_key
LLM_MODEL=gpt-4
EOF

# 3. Start all services
docker-compose -f docker-compose.mt5.yml up -d

# 4. Check logs
docker-compose -f docker-compose.mt5.yml logs -f trading-bot

# 5. Open dashboard
open http://localhost:8088
```

### Local Development

```bash
# Install dependencies
pip install -r requirements-mt5.txt

# Run bot directly
python -m freqtrade.mt5_llm.engine --config user_data/mt5_llm/config.json

# Run dashboard (separate terminal)
python -m freqtrade.mt5_llm.dashboard --port 8088
```

---

## Monitoring & Debugging

### Logs
- Bot logs: `docker logs mt5-trading-bot` or `user_data/mt5_llm/logs/bot.log`
- MT5 API logs: `docker logs mt5rest`
- Dashboard logs: `docker logs mt5-dashboard`

### Database Inspection
```bash
# Connect to SQLite
sqlite3 user_data/mt5_llm/mt5_bot.db

# View recent decisions
SELECT timestamp, action, confidence, reasoning FROM iterations ORDER BY timestamp DESC LIMIT 10;

# View trades
SELECT * FROM trades ORDER BY open_time DESC LIMIT 10;
```

### Health Checks
- MT5 API: `curl http://localhost:5015/Ping`
- Bot: `docker ps` (should show all containers running)
- Dashboard: `curl http://localhost:8088/api/status`

---

## Conclusion

This PRD outlines a pragmatic approach to building an autonomous LLM-powered trading bot for a hackathon. By leveraging proven patterns from Freqtrade while avoiding unnecessary complexity, we can deliver a working prototype in ~10 days that:

1. Trades BTC autonomously via MT5
2. Uses LLM reasoning for decisions
3. Enforces strict risk controls
4. Provides real-time monitoring
5. Runs reliably on Docker

The hybrid strategy (indicators + LLM + risk guardrails) balances innovation with safety, and the modular architecture allows for future enhancements without major rewrites.

**Next Steps:**
1. Review and approve this PRD
2. Set up MT5 demo account and LLM gateway
3. Begin Phase 1 implementation
4. Daily standups to track progress and blockers
