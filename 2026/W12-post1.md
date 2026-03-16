# W12 Post 1 — Auto-anchor recalibration: when your grid bot chases a trending market

> Grid trading works beautifully in sideways markets. It quietly breaks in trending ones — not with an error, but with silence.

---

## The Problem With Static Anchors

A grid bot works by placing buy orders below the current price and sell orders above it, at fixed intervals. The pivot point of that structure is the **anchor** — the reference price around which the grid is centered.

Set the anchor once and forget it. That's the standard advice.

It works fine until the market moves.

ETH spent several days consolidating around $2,200. My three grid bots (Arbitrum, Base, Linea) were anchored around $2,200. Then this morning, ETH ran to $2,255 — a ~2.5% move in a few hours.

From the bot's perspective, nothing broke. No errors. No stop-loss. It just... stopped being useful. When the price exceeds the top of your grid, you have no more sell orders to place. The bot sits at level 7 out of 8, waiting for a retracement that may or may not come, while the market keeps moving up.

The grid became one-sided. Buy-only. Effectively idle.

---

## Manual Recalibration: Works Once, Doesn't Scale

The immediate fix is simple: update the `gridAnchor` value in the state file, reset `lastLevel` to the center, and let the bot resume from a balanced position.

I did this manually this morning. Took about 2 minutes.

But *manual* means I need to notice the drift, decide to act, and do it — all within a window where it still matters. During European work hours, that's fine. At 3 AM on a Tuesday when ETH gaps up 5%, I'm asleep.

The real fix is automation.

---

## Design: What "Effective Recalibration" Actually Requires

The naive solution is obvious: if price drifts from anchor, move the anchor. But naive automation introduces its own problems.

**Problem 1: Chasing spikes**
If ETH flashes up 4% in 20 minutes and you immediately recenter, you've just anchored at the top of a spike. Price reverses, and now your grid is buying all the way down into a position you didn't want.

**Problem 2: Thrashing**
Recalibrate too eagerly, and the bot spends more time recalibrating than trading. Every recalibration is a reset — you lose the accumulated level history and start fresh.

**Problem 3: Which deviation is signal vs. noise?**
At 1% grid spacing and 8 levels, the grid covers ±4% from anchor. A 1% drift is noise. A 3% drift is structural — the center of gravity has shifted.

---

## The Solution: threshold + two guardrails

The logic I implemented in `auto_anchor.py` (runs every 15 minutes via cron):

**Trigger**: recalibrate when `|currentLevel - centerLevel| >= 2`

With 8 levels and 1% spacing, this means the price has moved at least 2% from the anchor — far enough that the grid is meaningfully asymmetric, not just bouncing.

**Guardrail 1 — Cooldown**: maximum one recalibration per bot every 4 hours.

Prevents cascading resets during volatile sessions. If the market is thrashing ±3% over two hours, we don't want to follow it tick by tick.

**Guardrail 2 — Spike filter**: skip recalibration if the 1-hour price change exceeds 3%.

A sharp 1-hour move is more likely a spike than a trend. We wait for the next cycle to confirm the new level is structural.

**Action**: `newAnchor = round(currentPrice / 10) * 10`, `lastLevel = centerLevel`

Rounding to the nearest $10 gives clean grid levels and avoids floating-point anchors that look like `$2,257.35`.

---

## Implementation Details

The script is intentionally isolated from the bot logic — it reads and writes the same state files the bots use, but runs independently. No changes to the trading scripts themselves.

```python
def maybe_recalibrate(name, cfg, state, eth_price, now_ts, last_recal_ts):
    spacing_pct  = float(cfg.get("gridSpacingPct", 1.0))
    grid_levels  = int(cfg.get("gridLevels", 8))
    center_level = grid_levels // 2

    anchor = float(state.get("gridAnchor", eth_price))
    levels = generate_grid_levels(anchor, spacing_pct, grid_levels)
    current_level = find_level_index(eth_price, levels)
    deviation = abs(current_level - center_level)

    cooldown_ok = (now_ts - last_recal_ts) >= 4 * 3600

    if deviation >= 2 and cooldown_ok:
        state["gridAnchor"] = round(eth_price / 10) * 10
        state["lastLevel"]  = center_level
        return True
    return False
```

The price history for spike detection is stored in a small JSON sidecar (`auto_anchor_state.json`) — a rolling buffer of the last ~15 price samples (15 min × 15 = ~3.75 hours of coverage). No external dependency, no database.

---

## Why Not Bake This Into the Bot?

I considered it. The counter-argument: the bot's job is to trade. The recalibrator's job is to manage the grid geometry. Keeping them separate means:

- I can disable recalibration independently without touching the trading logic
- Failures in the recalibrator don't affect trade execution
- The recalibrator can be tested, replaced, or extended without a bot restart
- Logs are clean and separated

The separation of concerns is worth the extra file.

---

## What This Doesn't Solve

Auto-recalibration keeps the grid centered. It doesn't solve the underlying tension in grid trading: **you're always selling your winners early and accumulating losers.**

In a sustained bull run, the bot will keep selling ETH as it climbs — profiting on each trade, but underperforming a simple hold. In a sustained downtrend, it will keep buying ETH as it falls — accumulating a position that bleeds value even as the bot reports "trades executed."

Grid trading extracts value from volatility. In a trending market, it's just a sophisticated way to be wrong slowly.

The recalibrator doesn't fix this. It just makes sure the bot stays in the game long enough to catch the next sideways phase.

---

## Current State

Three grid bots recalibrated to the current price this morning (manually, before the automation was in place). The auto-recalibrator is now live — next trigger will fire autonomously when price deviates enough.

The Hyperliquid short that was running as a hedge hit its stop-loss during the same rally. That position is closed. Whether to reopen it depends on funding rate conditions — a separate decision, not automated.

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
