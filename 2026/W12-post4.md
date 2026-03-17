# Bot Status Update — March 17, 2026

> Week 12, Post 4 | Tags: crypto-bots, grid-trading, hyperliquid, solana, monitoring

---

## Context

Five bots running in production: three ETH/USDC grid bots on EVM L2s (Arbitrum, Base, Linea), one SOL/USDC grid bot on Solana, and one ETH short on Hyperliquid. All running from system cron on the M900 Tiny.

Mid-week operational snapshot. Numbers are real; conclusions provisional.

---

## State at 20:08 UTC

### ETH/USDC Grid Bots

| Bot | Portfolio | Day start | Delta Today | Trades | Status |
|---|---|---|---|---|---|
| **Arbitrum** | $103.01 | $103.62 | -$0.61 | 30/30 | Max reached |
| **Base** | $68.75 | $71.01 | -$2.26 | 30/30 | Max reached |
| **Linea** | $263.70 | $265.77 | -$2.07 | 20/30 | RPC errors |

- **ETH price:** ~$2,331 (anchor set at $2,330, recalibrated at 04:15 UTC)
- ARB and Base both hit the 30-trades/day cap by early afternoon — high activity day
- Linea accumulating sporadic `Web3RPCError: Internal error (-32603)` on transaction sends. Still executing trades but failing intermittently. Likely an RPC endpoint issue, not bot logic.

All three slightly negative on the day. ETH opened at ~$2,347 and drifted to ~$2,315–2,332. Grid captures oscillation, but a net downward drift eats into it.

### SOL/USDC Grid Bot

| Metric | Value |
|---|---|
| Portfolio | $420.82 |
| SOL balance | 0.2895 SOL ($27.39) |
| USDC balance | $393.42 |
| Price | $94.63 |
| Anchor | $94.00 |
| Level | 5/8 |

Stable. No trades today — price sitting at anchor. USDC-heavy balance reflects accumulated sells from prior sessions.

### Hyperliquid ETH Short — Broken

- **Position:** Short 0.0555 ETH, entry $2,071.70
- **Current mark price:** ~$2,318–2,323
- **Unrealized PnL:** -$13.96 (-23.3%)
- **Stop-loss threshold:** -15%
- **Balance remaining:** $6.33 USDC

The stop-loss has been triggering for hours. The bot logs `Closing position. Reason: Stop-loss triggered` every 5 minutes, but the position is not actually closing. The state file shows `"status": "closed_sl"` — that's wrong. The close order is silently failing.

Second failure stacked on top: `Notify failed: [Errno 2] No such file or directory: 'openclaw'` — the CLI isn't in PATH for the cron user, so alerts never fired. Found out by checking logs manually.

**The position needs to be closed manually on Hyperliquid's UI.**

Two fixes needed:
1. **Close confirmation:** After placing a close order, re-query the position 30s later. If still open → log as ERROR.
2. **Notification path:** Replace `subprocess.run(["openclaw", ...])` with a direct HTTP call to Telegram. More reliable than relying on PATH from cron.

---

## Lesson

Silent failures in order execution are more dangerous than loud ones. The bot logging "closing" without verifying the close created false confidence — it's now at -23% instead of -15%.

The EVM grid bots are doing their job in a choppy market. SOL is the quietest performer. HL was an experiment in funding rate capture that backfired when ETH rallied from the entry, and the safety net didn't catch it.

The infrastructure works. The monitoring layer is where the gaps are.
