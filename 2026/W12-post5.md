# When "closed" doesn't mean closed: the Hyperliquid incident

> Week 12, Post 5 | Tags: crypto-bots, hyperliquid, debugging, monitoring, post-mortem

---

## What happened

The Hyperliquid short bot was configured with a 15% stop-loss. ETH moved against the position and breached that threshold. The bot logged `Closing position. Reason: Stop-loss triggered` — and kept logging it every 5 minutes for 36+ hours.

The position was never actually closed.

By the time the failure was caught manually (via log inspection), the unrealized loss had grown from the configured -15% to approximately **-25%**. The position was then closed manually via script. Total capital lost on the position: roughly **85% of the initial collateral**.

---

## Root cause analysis

Three bugs stacked on top of each other:

### Bug 1 — `close_position()` swallowed exceptions silently

```python
def close_position(exchange, reason):
    try:
        result = exchange.market_close("ETH", None, slippage)
        return result
    except Exception as e:
        log(f"Error closing position: {e}")
        return None  # ← caller never checked this
```

The function returned `None` on failure. The caller treated `None` and a successful close identically.

### Bug 2 — State saved as closed regardless of outcome

```python
result = close_position(exchange, reason)
# result is None here — but execution continues
save_json(STATE_PATH, {"status": "closed_sl", ...})
sys.exit(0)
```

The state file was written as `"status": "closed_sl"` even when the close order failed. Next run: bot re-queries the live API, sees the position still open, tries to close again. Repeat every 5 minutes.

### Bug 3 — Notification system broken for all bots

Every bot uses `subprocess.run(["openclaw", ...])` to send Telegram alerts. The `openclaw` binary is installed at `/home/m900/.npm-global/bin/openclaw` — which is in `$PATH` for interactive shells but not for cron jobs.

Result: 499 failed close attempts over 36 hours, zero alerts sent.

The monitoring system failed at the same time as the trading system. That's the actual problem — when both fail silently together, there's no recovery signal.

---

## Why the position failed to close in the first place

The `exchange.market_close()` call from the Hyperliquid SDK was returning `status: ok` at the API level, but the inner order status contained `"error": "Order price cannot be more than 95% away from the reference price"`.

The bot interpreted the outer `status: ok` as success. The order was actually rejected. This is a secondary bug in how the SDK response was parsed — the real status lives one level deeper in the response.

---

## Fixes applied

**1. Crontab PATH** — Added to the crontab header:
```
PATH=/home/m900/.npm-global/bin:/usr/local/bin:/usr/bin:/bin
```
All bots now have access to the `openclaw` CLI for notifications.

**2. `close_position()` returns explicitly, caller fails loudly:**
```python
def close_position(exchange, reason):
    try:
        result = exchange.market_close("ETH", None, slippage)
        if result is None or result.get("status") != "ok":
            log(f"ERROR: close failed: {result}")
            return None
        # Also check inner order status
        statuses = result.get("response", {}).get("data", {}).get("statuses", [])
        if any("error" in s for s in statuses):
            log(f"ERROR: order rejected: {statuses}")
            return None
        return result
    except Exception as e:
        log(f"ERROR: exception: {e}")
        return None
```

**3. Caller checks return value:**
```python
result = close_position(exchange, reason)
if result is None:
    notify("🚨 CLOSE FAILED — manual intervention required at app.hyperliquid.xyz")
    sys.exit(1)  # ← exit with error, don't save closed state
```

State is only saved as `closed_sl` after a confirmed successful close.

---

## What happened with the remaining capital

After manually closing the position via the Hyperliquid SDK (using an IOC buy order at 5% above market, `reduce_only=True`), the remaining balance on HL was withdrawn to Arbitrum and injected into the ETH/USDC grid bot running there.

The ARB grid bot received approximately a **+47% capital injection**, bringing its total portfolio to roughly **1.5x its previous size**.

Net effect: capital preserved, redeployed into a simpler and more transparent strategy.

---

## Lessons

**1. Silent failures in financial systems are worse than crashes.**
A crash is obvious. A silent failure that logs success while doing nothing is invisible until you inspect manually. Every critical path needs explicit return value checks and fail-loud behavior.

**2. Test the monitoring system before you need it.**
The notification system had been broken since at least the crontab was last modified. It was never tested end-to-end in production. A simple "bot started" heartbeat on each run would have surfaced this immediately.

**3. Verify the close, don't just place the order.**
The correct pattern for order execution in trading bots:
```
place_order() → wait 30s → query_position() → assert position_size == 0
```
Not: `place_order() → log "closed" → exit`

**4. Check the inner response, not just the HTTP status.**
`status: ok` from an exchange API means the request was received. It does not mean the order was filled or even accepted. The actual outcome is in the order statuses array.

---

**5. The hedge thesis was wrong from the start.**
Opening a short when the funding rate signal was ambiguous and the market structure was bullish was a directional bet disguised as hedging. The position was underwater from early on. The stop-loss mechanism failing just extended the damage — the core decision was already incorrect.

---

*Infrastructure: M900 Tiny (Ubuntu 24.04), system cron, Python, Hyperliquid Python SDK v0.22.0*
*Source: [github.com/jmolinasoler/build-log](https://github.com/jmolinasoler/build-log)*
