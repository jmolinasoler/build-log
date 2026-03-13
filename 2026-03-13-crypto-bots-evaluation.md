# Crypto Bots Evaluation - March 13, 2026

## Overview

Running automated grid trading bots across three L2 networks (Arbitrum, Base, Linea) plus a hedging position on Hyperliquid. This is a technical evaluation of performance, configuration, and lessons learned after ~1 month of live trading.

## Bot Configuration

### Grid Trading Strategy
- **Grid Spacing:** 2.5% (adjusted from initial 1.5% to reduce overtrading)
- **Grid Levels:** 8
- **Max Trades/Day:** 30
- **Stop Loss:** 15%

### Active Bots

| Bot | Network | Trade Size | Status | Trades Today | Last Trade |
|-----|---------|------------|--------|--------------|------------|
| **Arbitrum Grid** | Arbitrum One | $14/trade | Active | 30/30 | 14:25 UTC |
| **Base Grid** | Base | $5/trade | Active | 20/30 | 15:25 UTC |
| **Linea Grid** | Linea | $8/trade | Active | 13/30 | 15:40 UTC |
| **Hyperliquid Perp** | Hyperliquid | 2x leverage | Open | - | Position monitoring |

## Current State (15:50 UTC)

### Arbitrum Grid Bot
- **Grid Anchor:** $2,169
- **Last Price:** $2,197.71 (ETH)
- **Position:** Level 5 (above anchor, bias toward selling)
- **Trades Today:** 30 (limit reached)
- **Initial Portfolio:** $108.13 USDC
- **Current Balance:** ~$90.49 USDC + 0.0065 ETH (~$14.27) = **~$104.76 total**
- **P&L:** ~-$3.37 (-3.1%)

### Base Grid Bot
- **Grid Anchor:** $2,159
- **Last Price:** $2,150.83 (ETH)
- **Position:** Level 3 (near anchor, balanced)
- **Trades Today:** 20/30
- **Initial Portfolio:** $38.79 USDC
- **Current Balance:** ~$71.08 USDC + 0.0032 ETH (~$6.87) = **~$77.95 total**
- **P&L:** ~+$39.16 (+101%)

**Note:** This exceptional performance on Base suggests either:
1. Additional deposits were made (not tracked in initialPortfolioUsdc)
2. The bot started with a different balance than recorded
3. Extremely favorable market conditions for this specific grid

Need to verify the actual deposit history.

### Linea Grid Bot
- **Grid Anchor:** $2,169
- **Last Price:** $2,152.35 (ETH)
- **Position:** Level 3 (below anchor, bias toward buying)
- **Trades Today:** 13/30
- **Initial Portfolio:** $118.60 USDC
- **Current Balance:** ~$162.71 USDC + 0.0458 ETH (~$98.58) = **~$261.29 total**
- **P&L:** ~+$142.69 (+120%)

**Note:** Similar to Base, this performance suggests additional deposits or inaccurate initial tracking.

### Hyperliquid Perp Bot (ETH Short 2x)
- **Entry Price:** $2,071.70
- **Mark Price:** $2,140.10
- **Position Size:** -0.0555 ETH (short)
- **Collateral:** $57.46
- **Leverage:** 2x
- **Unrealized P&L:** **-$3.80 (-6.6% of collateral)**
- **Funding Earned:** -$0.15 (net negative)
- **Current Funding Rate:** +0.0023% per 8h (+0.0069%/day, +2.5%/year)
- **Liquidation Price:** $3,048.74
- **Position Opened:** Feb 25, 2026 (17 days ago)

**Analysis:** The short position is underwater because ETH rallied from $2,071 → $2,140 (+3.3%). Funding rates have been positive (paying to be short), adding to losses. The hedge hasn't performed as intended in this bull move.

## Technical Notes

### What's Working
- **Grid spacing adjustment (2.5%):** Reduced noise trades while still capturing meaningful moves
- **Daily trade limits:** Prevents runaway execution in volatile conditions
- **Multi-chain deployment:** Diversifies execution risk and gas costs

### What's Not Working
- **Short hedge timing:** Opened the HL short right before a rally
- **Portfolio tracking:** Initial balance discrepancies make P&L analysis unreliable
- **Funding rate assumptions:** Expected negative funding (getting paid to short), but it's been positive

### Observed Behavior
- **Arbitrum:** Hit the 30-trade daily limit, suggesting high volatility or tight grid spacing relative to market movement
- **Base & Linea:** More moderate activity (20 and 13 trades), indicating either wider effective spreads or lower volatility
- **Trade clustering:** Recent trades show oscillation patterns (buy-sell-buy-sell), which is expected behavior for grid trading

## Gas & Execution Costs

| Network | Avg Gas/Trade | Total Gas Today | Efficiency |
|---------|---------------|-----------------|------------|
| Arbitrum | ~$0.05 | ~$1.50 (30 trades) | High |
| Base | ~$0.02 | ~$0.40 (20 trades) | Very High |
| Linea | ~$0.03 | ~$0.39 (13 trades) | Very High |

**Total gas spent today:** ~$2.29

L2 gas costs remain negligible compared to trade sizes ($5-14), making frequent grid trading economically viable.

## Lessons Learned

1. **Track deposits separately:** The initial portfolio values are unreliable without separate deposit tracking
2. **Funding rates are not static:** The HL short assumed negative funding (profit), but positive funding (cost) has been persistent
3. **Grid anchors need recalibration:** When price moves significantly from anchor, the grid becomes asymmetric (e.g., Arbitrum at level 5)
4. **Daily trade limits are essential:** Without them, Arbitrum would have executed far more trades in high volatility

## Next Steps

1. **Implement proper accounting:** Track deposits/withdrawals separately from trading P&L
2. **Reanchor grids:** When price deviates >3 levels from anchor, recalibrate to current market
3. **Evaluate HL short:** Consider closing if funding remains positive and position stays underwater
4. **Monitor ETH volatility:** If market enters low-volatility range trading, these grid bots should perform well

## System Reliability

- **Uptime:** 100% (cron-based execution every 5 minutes)
- **Failed trades:** 0 (all transactions confirmed on-chain)
- **Errors:** None logged in recent runs
- **Node connectivity:** Helios light client + Ankr RPC backup working smoothly

## Conclusion

The grid trading strategy is executing as designed, but **performance evaluation is blocked by accounting issues**. The Arbitrum bot shows a small loss, which is expected in a trending market (grid trading profits in range-bound conditions). The Base and Linea bots show unrealistic gains, indicating tracking errors.

The Hyperliquid short hedge has not performed as intended—it's underwater and paying funding rather than earning it. This highlights the risk of directional bets as "hedges."

**Overall assessment:** The infrastructure is solid, but strategy refinement and proper accounting are needed before making capital allocation decisions.

---

**Tech Stack:**
- Python (web3.py for EVM, hyperliquid-python-sdk)
- System cron (5-min intervals, no AI token cost)
- Helios (Ethereum light client for trustless RPC)
- Arbitrum, Base, Linea, Hyperliquid

**Repository:** Private (not published)  
**Monitoring:** Manual via state files + daily logs
