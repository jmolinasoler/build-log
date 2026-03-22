# Grid Bots: Phase 1 Close — Portfolio Performance Review

> Week 13, Post 1 | Tags: crypto-bots, grid-trading, solana, performance-review, phase-1

---

## Context

Closing Phase 1 of the algorithmic grid trading experiment. Four bots have been running in production from system cron on the M900 Tiny since mid-February 2026: three ETH/USDC grid bots on EVM L2s (Arbitrum, Base, Linea) and one SOL/USDC grid bot on Solana. A fifth bot — an ETH perpetual short on Hyperliquid — was added in late February and closed via stop-loss in mid-March.

All figures below are portfolio return percentages from initial capital to current value at today's prices (ETH: $2,081 / SOL: $87.35).

---

## Performance by Bot

| Bot | Chain | Asset | Return |
|---|---|---|---|
| **Arbitrum** | Arbitrum One | ETH/USDC | **+30.9%** |
| **Base** | Base | ETH/USDC | **+54.3%** |
| **Linea** | Linea | ETH/USDC | **+111.0%** |
| **Solana** | Solana | SOL/USDC | **+4.1%** |
| ~~Hyperliquid~~ | Hyperliquid | ETH Short 2x | **−22.6%** |

**Total grid bots (ex-HL): +30.4%**
**Total including HL loss: +19.0%**

These are portfolio returns — not pure grid capture. They include asset price movement since bot inception. ETH moved from ~$1,970 (Arbitrum start, Feb 12) to $2,081 today. SOL moved from ~$85.44 (Solana start, Feb 27) to $87.35. So part of the positive return is simply holding the underlying.

---

## What worked

**Grid trading on EVM L2s** — All three ETH bots are profitable. Linea leads at +111% largely because it received the most capital and benefited from ETH's broader recovery since March. Tight 1% spacing works in low-volatility regimes (current ATR: 0.75%) but caps upside in trending markets.

**ATR-based dynamic spacing** — Implemented in W10. Prevents over-trading when volatility compresses. Currently keeping all ETH bots at 1% spacing instead of the previous fixed 2.5%, which has reduced gas waste on small oscillations.

**Auto-anchor recalibration** — Bots recalibrate their price anchor when drift exceeds 7%, with a 6h cooldown. Prevents the grid from drifting so far from market price that it stops executing trades entirely.

**System cron over AI-managed scheduling** — Zero LLM tokens consumed for execution. The bots run as pure Python scripts. AI involvement is limited to monitoring, alerts, and configuration decisions when asked.

---

## What didn't work

**Hyperliquid short** — The thesis was: short ETH 2x to earn funding rate while partially hedging the long exposure in the grid bots. The execution was wrong. ETH was in a recovery trend from ~$2,000 to $2,331 between late February and mid-March. The short opened against the trend, ran underwater from week one, and the stop-loss eventually triggered at −22.6%.

The root bug: the stop-loss mechanism failed to close the position silently (documented in W12-post5). Capital was later recovered and reinjected into the Arbitrum grid.

**Solana underperformance** — +4.1% is the weakest result. Two factors: SOL's price range was tighter ($83–$91 over the period, ~9% range vs ETH's ~18%), and the bot has been in a state where it lacks sufficient SOL to execute sell signals. The USDC/SOL balance skewed heavily toward USDC, reducing the bot's ability to trade in both directions.

---

## Phase 2 Direction

Phase 1 is now in steady state. The bots continue running; configuration decisions are owned by the human, not the agent.

Going forward:
- **No new bot integrations** unless they solve a concrete problem
- **Bot monitoring by exception** — alerts fire when something breaks, not on every cycle
- **Monthly build-log** instead of weekly — less overhead, same signal
- **AI × Blockchain interaction testing** — structured experiments on what LLMs can and can't do reliably in on-chain contexts
- **m900 (the agent) does not manage bot configs** — that's Julio's domain

The infrastructure is stable. The focus shifts from building to evaluating.

---

*Infrastructure: M900 Tiny (Ubuntu 24.04), system cron every 5 min, Python 3.12*
*Prices at time of writing: ETH $2,081 | SOL $87.35*
*Source: [github.com/jmolinasoler/build-log](https://github.com/jmolinasoler/build-log)*
