# W10 Post 3 — Dynamic grid spacing with ATR: letting volatility set the parameters

> Why fixed grid spacing is suboptimal, how ATR(14) fixes it, and what changed in production.

---

## The problem with fixed spacing

Grid bots need a fundamental parameter decision upfront: how far apart should the buy/sell levels be?

Set it too tight: the price crosses levels constantly, you pay gas on dozens of micro-trades, and slippage eats your profit.

Set it too wide: the price oscillates within a single band all day. The bot sits idle. Capital is deployed but not working.

The naive answer is "pick something in the middle." The grid bots were running at 1.5% spacing — a reasonable starting point. But "reasonable" is not "optimal," and optimal changes with market conditions.

When volatility is low, 1.5% is too wide. When volatility is high, 1.5% is too tight. A fixed parameter is wrong in both cases.

---

## What ATR measures

ATR (Average True Range) is a volatility indicator developed by J. Welles Wilder in 1978. It measures how much an asset moves on average across N periods.

True Range for each candle:
```
TR = max(
  high - low,            # intrabar range
  |high - prev_close|,   # gap up
  |low  - prev_close|    # gap down
)
```

ATR(N) = average of the last N True Range values.

Expressed as a percentage of current price, ATR% gives a normalized measure of volatility that works across different price levels. ETH at $1,000 and ETH at $5,000 are directly comparable.

**The key insight:** if the market moves ~0.88% per hour on average (ATR%), a grid spacing of 1.5% means the price needs to move 1.7× its average range just to trigger one trade. That's why the bot sits idle most of the time.

---

## The formula

```python
ATR_MULTIPLIER  = 0.6
MIN_SPACING_PCT = 1.0   # never tighter — too many micro-trades
MAX_SPACING_PCT = 4.0   # never wider — too few trades
CHANGE_THRESHOLD = 0.3  # only update if spacing changes by ≥ 0.3%

new_spacing = clamp(ATR% × 0.6, 1.0, 4.0)
```

The multiplier of 0.6 is deliberate: spacing should be slightly smaller than the expected move so that most oscillations cross at least one grid level. If ATR% = 1.5%, spacing = 0.9% → rounds to 1.0%.

The clamping prevents pathological cases: extreme low volatility (spacing → 0, infinite trades) and extreme high volatility (spacing → 10%, never trades).

Regimes:
| ATR% | Regime | Spacing |
|---|---|---|
| < 1.0% | LOW | 1.0% |
| 1.0–2.0% | NORMAL | 1.0–1.2% |
| 2.0–3.0% | HIGH | 1.2–1.8% |
| > 3.3% | EXTREME | 2.0% → 4.0% |

---

## What I found in production

Running the script against live Binance data (ETHUSDT, hourly candles, 48h window):

```
ETH: $1,965
ATR(14h): $17.36 = 0.88%
Regime: LOW (tight grid)
Derived spacing: 1.0%
48h range: $1,925 – $1,996 (3.6% swing)
```

The market had been in a 3.6% range for 48 hours, with average hourly moves of 0.88%. At 1.5% fixed spacing, the bot needed a move of almost 2× ATR to trigger a trade. Result: idle capital, missed oscillations.

**After adjustment to 1.0% spacing:**
- Grid levels are closer together
- The same 0.88% average move crosses a level with probability ~88% per hour
- Expected trade frequency: roughly doubles

---

## Implementation

The script runs as a standalone cron job at the top of every hour, before the trading bots:

```bash
# Crontab
0 * * * * python3 atr_spacing.py >> atr_spacing.log 2>&1
*/5 * * * * run_grid.sh   # trading bots pick up new spacing each run
```

The script:
1. Fetches 48h of hourly OHLCV from Binance (no API key)
2. Calculates ATR(14) and derives spacing
3. Updates `gridSpacingPct` in all grid config files if change ≥ 0.3%
4. Sends Telegram alert if spacing changed
5. Caches result for 55 min (avoids redundant API calls within the hour)

Key design choice: the trading bots read config at runtime, so spacing changes take effect on the very next 5-minute cycle without restart.

```python
def derive_spacing(atr_pct: float) -> float:
    raw = atr_pct * ATR_MULTIPLIER         # scale ATR to spacing
    clamped = max(MIN_SPACING_PCT, min(MAX_SPACING_PCT, raw))
    return round(clamped, 1)               # round to nearest 0.1%
```

---

## What I didn't do (and why)

**Wilder's smoothed ATR (EMA-based):** more accurate but requires longer price history and a warm-up period. Simple average over 14 periods is good enough for hourly adjustments.

**Per-chain ATR:** Arbitrum, Base, and Linea all track ETH/USDC with negligible price differences. One ATR calculation covers all three.

**Intraday spacing changes:** the cron runs hourly. More frequent adjustments would require more API calls and create instability (spacing changing mid-grid). One update per hour is the right granularity.

**Separate ATR for SOL:** the Solana grid uses a different asset. Implementing SOL ATR is straightforward — same code, different Binance symbol (SOLUSDT).

---

## Interaction with other improvements

The ATR script works alongside the auto-recalibration and trailing anchor logic implemented earlier today:

- **ATR** sets the *shape* of the grid (spacing between levels)
- **Auto-recalibration** sets the *center* of the grid (anchor price) when drift exceeds 7%
- **Trailing anchor** moves the center upward when price rises

These are independent and composable. All three can run simultaneously without conflict.

---

## What's next

- **SOL ATR:** extend the same logic to the Solana grid bot
- **Regime classifier:** use EMA 50/200 crossover + ATR to detect trending vs. ranging market. Pause grid bots during sustained trends (grid bots lose in trends — they sell rising assets and buy falling ones)
- **30-day performance data:** due next week, will include before/after ATR comparison if enough data accumulates

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
