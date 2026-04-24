# L-Shape Live Trading System - Handoff Document

Spinoff from `btc-lshape-long/live-trading` → standalone `lshape-long-live` repository.

---

## Project Overview

ETHUSDT L-Shape pattern live trading system with Go engine + Svelte frontend, containerized with Podman.

### Architecture

```
lshape-long-live/
├── engine/              # Go trading engine
│   ├── cmd/main.go      # Entry point
│   └── internal/
│       ├── config/      # Strategy profiles (5m-balanced, 5m-aggressive, 15m-aggressive)
│       ├── detector/    # Enhanced L-shape detection (ATR-based)
│       ├── engine/      # Trading logic, position/trade management
│       ├── exchange/    # Binance Futures API client
│       ├── notifier/    # Telegram notifications
│       ├── server/      # HTTP API server
│       └── types/       # Shared data types
├── frontend/            # Svelte SPA
│   └── src/
│       ├── lib/         # API client, config
│       └── routes/      # Dashboard, History, Settings
├── Dockerfile           # Multi-stage build (Node + Go + Alpine)
├── podman-compose.yml   # Container orchestration
├── build.sh             # Build container
├── start.sh             # Start container
├── stop.sh              # Stop container
├── .env.dev             # Development/testnet config
└── .env.prod            # Production config
```

---

## Strategy Profiles (Backtest Verified)

| Profile | Position | Leverage | TP | SL | HC | CAGR | MDD | Calmar |
|---------|:--------:|:--------:|:--:|:--:|:--:|:----:|:---:|:------:|
| **5m-balanced** | 20% | 10x | 15% | 2% | 5% | 33.4% | 32.0% | **1.05** |
| 5m-aggressive | 30% | 10x | 15% | 2% | 5% | 46.8% | 46.3% | 1.01 |
| 15m-aggressive | 30% | 10x | 15% | 2% | 5% | 60.7% | 69.9% | 0.87 |

**Recommended**: `5m-balanced` (Best risk-adjusted returns)

---

## L-Shape Detection Logic

ATR-based dynamic thresholds matching backtest SOTA:

| Timeframe | DropATR | ConsolATR | Flatness | MinConfidence |
|:---------:|:-------:|:---------:|:--------:|:-------------:|
| 5m | 1.5 | 1.0 | 0.5 | 0.5 |
| 15m | 2.0 | 1.2 | 0.45 | 0.5 |
| 1h | 2.5 | 1.5 | 0.4 | 0.6 |

**Entry Conditions** (all must pass):
1. Prior drop: maxHigh(first half) → minLow(second half) ≥ ATR% × DropATRMultiplier
2. Consolidation: range% ≤ ATR% × ConsolATRMultiplier AND flatness ≤ threshold
3. MA50 breakout: prevClose < prevMA AND currClose > currMA AND bullish candle
4. Confidence ≥ MinConfidence

**Exit Conditions**:
- Stop Loss: -2% from entry
- Half Close: +5% → sell 50%, move SL to breakeven
- Take Profit: +15%

---

## Environment Variables

```bash
# Exchange
BINANCE_API_KEY=xxx
BINANCE_SECRET_KEY=xxx
SYMBOL=ETHUSDT
TESTNET=true|false

# Strategy
STRATEGY_PROFILE=5m-balanced
TRADING_ENABLED=false

# Telegram
TELEGRAM_BOT_TOKEN=xxx
TELEGRAM_CHAT_ID=xxx
TELEGRAM_ENABLED=false

# Server
HOST_PORT=8088
```

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/status` | GET | Current status (trading on/off, position, trade count) |
| `/api/config` | GET | Config info (profile, leverage, TP/SL) |
| `/api/config` | POST | Update config (position_size) |
| `/api/position` | GET | Current position details |
| `/api/trades` | GET | Trade history |
| `/api/toggle` | POST | Toggle trading on/off |
| `/api/telegram/toggle` | POST | Toggle Telegram notifications |
| `/api/profiles` | GET | Available profiles |

---

## Deployment

### Build & Run

```bash
# Development (testnet, port 8088)
./build.sh dev
./start.sh dev

# Production (mainnet, port 8080)
./build.sh prod
./start.sh prod

# Stop
./stop.sh

# Logs
podman-compose logs -f
```

### Telegram Setup

1. Create bot via @BotFather → get `TELEGRAM_BOT_TOKEN`
2. Start conversation with bot
3. Get chat_id: `https://api.telegram.org/bot<TOKEN>/getUpdates`
4. Add to `.env.dev` or `.env.prod`

---

## Backtest Reference

Full backtest results: https://mgcha85.github.io/btc-lshape-long/realistic-backtest

**Fee Assumptions**:
- Maker: 0.02%
- Taker: 0.05%
- Funding: ~0.01%/8h

**Data**: ETHUSDT 2020-2026

---

## Known Limitations

1. **Single symbol**: Currently hardcoded for ETHUSDT
2. **No persistence**: Trade history lost on container restart (volume mount for `/app/data` available but not implemented)
3. **API keys in env**: Consider secrets management for production

---

## Next Steps

1. Add trade history persistence (SQLite or file-based)
2. Add multiple symbol support
3. Add Prometheus metrics endpoint
4. Add WebSocket for real-time updates
5. Add backtesting mode for strategy validation

---

## Original Repository

Parent: https://github.com/mgcha85/btc-lshape-long

Backtest code: `src/crypto_backtest/detection/enhanced_rules/detector.py`
