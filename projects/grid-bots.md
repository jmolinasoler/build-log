# Algorithmic Grid Trading — Architecture & Lessons

> Status: 🔄 In production  
> Networks: Arbitrum, Base, Linea (EVM) + Solana  
> Infrastructure: Lenovo M900 Tiny, system cron (no cloud)  

---

## Overview

4 grid trading bots running in production on real capital. The goal: capture profit from price oscillation in sideways/bear markets without needing to predict direction.

**Grid trading in one sentence:** Place buy orders below current price and sell orders above it, in a ladder. When price bounces between levels, you accumulate small profits.

---

## Architecture

```
M900 Tiny (Ubuntu 24.04)
├── system cron (every 5 min)
│   ├── arb_grid_trader.py    — ETH/USDC, Arbitrum
│   ├── base_grid_trader.py   — ETH/USDC, Base
│   ├── linea_grid_trader.py  — ETH/USDC, Linea
│   ├── sol_grid_bot.py       — SOL/USDC, Solana
│   └── hl_perp_bot.py        — ETH short, Hyperliquid
└── OpenClaw agent (Telegram alerts, monitoring)
```

No cloud. No managed services. Cron + Python + on-chain transactions.

---

## What I Learned Shutting Down the Memecoin Bot

Before grid bots, I ran a Solana memecoin momentum bot for 2 weeks.

**Results:**
- Win rate: 14.3%
- Expectancy: −$18.2 per trade
- Total realized P&L: −$56.53

**Why it failed:** Bear market conditions. Momentum strategies need trend. In a sideways/down market, you buy breakouts that immediately reverse. The math doesn't lie — negative expectancy means you lose money at scale, regardless of occasional wins.

**Decision rule I applied:** If expectancy < 0 after sufficient sample, shut it down. No emotional attachment.

Grid bots are the logical alternative for the same market conditions.

---

## Parameters (current)

| Bot | Pair | Levels | Spacing | Trade size | Stop loss |
|---|---|---|---|---|---|
| Arbitrum | ETH/USDC | 8 | 1.5% | $14 | −15% |
| Base | ETH/USDC | 8 | 1.5% | $5 | −15% |
| Linea | ETH/USDC | 8 | 1.5% | $8 | −15% |
| Solana | SOL/USDC | 8 | 2.0% | $25 | −20% |

---

## Key Design Decisions

**Why multiple networks?**  
Gas costs on Ethereum mainnet make small grid trades unprofitable. L2s (Arbitrum, Base, Linea) have sub-cent transaction costs.

**Why a perp short alongside grid longs?**  
Grid bots are net-long (they accumulate the asset when price drops). A short position on Hyperliquid (2x leverage) provides partial hedge and earns funding rate when positive.

**Why system cron instead of a persistent process?**  
- Simpler. If the process crashes, cron restarts it automatically.
- Each run is stateless — state persisted in JSON files.
- Zero memory leak risk over weeks/months.

**Anchor recalibration:**  
When price moves significantly away from the grid anchor, the bot is either all-in or all-out of the base asset. Periodic recalibration resets the grid around the current price. This is a manual decision (not automated) — I evaluate it when price drifts >5%.

---

## Evaluation (30-day — pending)

Will publish performance data at the 30-day mark. Metrics:
- Total trades executed
- Win rate per bot
- Net P&L (including gas)
- Comparison: grid bot vs. simple hold
