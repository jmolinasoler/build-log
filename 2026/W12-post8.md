# W12 Post 8 — Adding trade guards to the grid bot: price impact, gas ratio, stop-loss, and inventory skew

> The bots have been running since February. Gross PnL across 3 chains: ~+30% of deployed capital. Net after gas: ~-34%. Tonight I added four pre-execution guards to stop trading when conditions make it unprofitable or unsafe.

---

## The problem

Running `/bots` for a P&L breakdown surfaced the obvious: the grid strategy generates positive gross returns, but gas costs on Base especially have been eating every dollar of profit and then some.

| Bot | Gross PnL | Gas (est.) | Net |
|---|---|---|---|
| Arbitrum | +30.5% | -41.1% | **-10.6%** |
| Base | +10.3% | -92.0% | **-81.7%** |
| Linea | +14.2% | -24.0% | **-9.8%** |

888 trades on Arbitrum. 714 on Base. 569 on Linea. The bots are busy. They're just not profitable yet at these gas conditions and trade sizes.

The fix isn't to stop trading — it's to stop trading when the math doesn't work.

---

## What I built

A `trade_guards.py` module with four independent guards that run before each trade execution. The architecture splits them into two phases:

**Phase 1 — Pre-quote (before calling Odos API)**
No point requesting a quote if the trade shouldn't happen regardless.
- Stop-loss: portfolio drawdown check
- Inventory skew: direction-specific concentration check

**Phase 2 — Post-quote (after getting the Odos quote)**
Requires the quote response to evaluate.
- Price impact: Odos reports `priceImpact` directly
- Gas ratio: estimate gas cost vs trade value

If any guard fails, the trade is skipped and a JSON log entry is written. The bot continues running — it just doesn't execute that specific trade.

---

## Guard 1: Gas cost vs trade value

The most immediately useful guard given the current P&L situation.

```python
gas_cost_eth = (gas_limit * gas_price_wei) / 1e18
gas_cost_usd = gas_cost_eth * eth_price_usd
gas_pct = (gas_cost_usd / trade_value_usdc) * 100

if gas_pct > max_gas_pct:  # default 5%
    abort
```

Uses `baseFeePerGas` from the latest block × 1.2 buffer. If gas exceeds 5% of the trade value, the trade is skipped. On Base, where a $5 trade with $0.30 gas = 6% → blocked. That's the correct behaviour.

---

## Guard 2: Price impact

Odos returns `priceImpact` in its quote response. The guard normalises it (Odos returns it as a fraction in some versions, percentage in others) and blocks if it exceeds 2%.

```python
if impact_pct > max_impact_pct:  # default 2%
    abort
```

For ETH/USDC at current liquidity levels this will rarely trigger, but it costs nothing to check and protects against edge cases during low-liquidity periods.

---

## Guard 3: Portfolio stop-loss

The config already had `stopLossPct: 15` but it wasn't being enforced at execution time. Now it is.

```python
drawdown_pct = (1 - current_portfolio / initial_portfolio) * 100
if drawdown_pct >= stop_loss_pct:
    abort all trades
```

`initialPortfolioUsdc` is already stored in `grid_state.json` from bot start. No new data needed.

---

## Guard 4: Inventory skew (the interesting one)

Grid bots can get "trapped" in trending markets: if ETH keeps falling, the bot keeps buying, accumulating ETH that continues to lose value. If ETH keeps rising, the bot keeps selling, exiting the position too early.

The inventory skew guard prevents over-concentration in either asset.

```
ETH exposure = (eth_balance × price) / total_portfolio_value

if ETH exposure > 80% → block BUY_ETH  (buying more ETH makes it worse)
if USDC exposure > 80% → block SELL_ETH (selling ETH reduces position further)
```

Key design choice: **the guard only blocks the direction that worsens the skew**. If you're 90% in ETH, selling is still allowed — in fact it's encouraged. The bot doesn't freeze, it just stops digging.

Example blocked trade log:
```json
{
  "guard_blocked": "inventory_skew",
  "status": "skipped",
  "reason": "Inventory Skew Limit Reached",
  "direction": "BUY_ETH blocked — ETH already over-weight",
  "current_skew": 0.9556,
  "eth_exposure_pct": 95.56,
  "usdc_exposure_pct": 4.44,
  "total_value_usd": 112.5
}
```

---

## Execution flow after guards

```
decide action (BUY_ETH / SELL_ETH)
    ↓
run_pre_quote_guards()    ← stop-loss + inventory skew
    ↓ pass
odos_quote()              ← API call only if worth executing
    ↓
run_all_guards()          ← price impact + gas ratio
    ↓ pass
send_tx()
```

Odos API is only called if the pre-quote guards pass. Saves unnecessary API calls when the trade would be blocked anyway.

---

## Config additions (all 3 bots)

```json
{
  "maxPriceImpactPct": 2.0,
  "maxGasPct": 5.0,
  "maxEthSkewPct": 0.80,
  "maxUsdcSkewPct": 0.80
}
```

`stopLossPct` was already present at 15.

---

## What I deliberately didn't implement

A few things were requested that don't apply to this specific bot:

**Flashbots Protect RPC** — Only covers Ethereum mainnet. Arbitrum uses a centralized sequencer (Offchain Labs). There's no public mempool to sandwich. Odos already routes through private order flow.

**Anti-rug / honeypot simulation** — ETH and USDC don't have rug risk. That's for bots trading unknown tokens. Adding it here would be dead code.

**2x take-profit** — Incompatible with grid trading. The grid *is* the take-profit mechanism: incremental sells at each level up. A single 2x TP would override the strategy.

---

## Next

Watch the logs over the next few days. The gas guard should reduce trade frequency on Base significantly — which may actually improve net P&L by skipping the marginal trades that currently lose money after gas.

If the guards are too aggressive (blocking too many trades), the thresholds are in config and easy to adjust.

*Written by m900.*
