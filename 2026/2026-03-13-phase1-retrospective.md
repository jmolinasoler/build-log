# Phase 1 Retrospective: AI Models, APIs, and the Cost of Figuring It Out

**Date:** 2026-03-13  
**Status:** Closing Phase 1 — AI-Assisted Local Infrastructure + Grid Trading  
**Next:** 1-year project with m900

---

## What We Closed

Two projects that ran in parallel since early February 2026 are now transitioning to steady state:

1. **AI-Assisted Local Infrastructure** — setting up m900 (Lenovo ThinkCentre M900 Tiny) as a persistent AI agent running OpenClaw, with memory, crons, and tool access to the real world.
2. **Algorithmic Grid Trading (EVM + Solana)** — deploying and tuning 4 grid bots (Arbitrum, Base, Linea, Solana + a Hyperliquid perp short) with ATR-based dynamic spacing.

Both worked. Neither worked cleanly.

---

## AI Model Comparison: What Actually Happened

Tested across multiple models over ~6 weeks:

| Model | Verdict |
|---|---|
| **Claude Sonnet 3.5 / 4** | Best reasoning + tool use. Default choice. |
| **Claude Opus** | Noticeably better on complex tasks. Expensive. |
| **Google Gemini** | Good. Already have the GCP environment — should use more. |
| **Venice AI** | Interesting privacy positioning, but contract quality is poor. Not reliable enough for production. |
| Others | Tested, discarded. |

**The real constraint isn't quality — it's queries per user license.** Claude Sonnet and Opus are the best tools, but a standard license throttles you fast if the agent is doing proactive work (heartbeats, crons, monitoring). The fix isn't switching models — it's diversifying so Claude handles reasoning-heavy tasks and Google handles volume or batch work.

**APIs cost money.** Spent ~€300 total across AI APIs (Google being the largest chunk). The lesson: APIs are fine for experimentation, not for ongoing agent overhead. The agent's background tasks should use zero-cost paths wherever possible — system cron + bash > OpenClaw cron + LLM for anything that doesn't require reasoning.

---

## Grid Bots: What the Data Says

4 bots live across 3 EVM chains + Solana. Summary at closing Phase 1:

- **Arbitrum Grid** — operational, ETH volatile in February, grid held
- **Base Grid** — smaller capital, stable
- **Linea Grid** — largest USDC position, profitable in sideways markets
- **Solana Grid** — added late, 18 trades/day average, currently +4.1%
- **Hyperliquid Short** — ETH perp 2x, partially hedges EVM grid exposure

ATR-based dynamic spacing (implemented W10) improved the bots' behavior in volatile periods — wider grids on high volatility, tighter on low. Running hourly.

**The decision going forward:** bots run on system cron, fully automated. The AI agent does not manage bot operations — that stays human-controlled to avoid overloading the agent with tasks that don't require reasoning.

---

## What Didn't Work

- **Google Coral TPU on M900** — hardware keying mismatch. The TPU requires M.2 NVMe slot, not WiFi slot. Physical blocker, not software.
- **VPS agent (Hetzner)** — ran a second agent for basketball scheduling. Deprecated. Unnecessary complexity.
- **Over-relying on AI for simple automation** — anything that can be a bash script should be a bash script. The agent's value is in judgment, not in running `df -h` on a schedule.

---

## The 1-Year Project: m900 + AI × Blockchain

Starting now. No rush, no sprints. One machine, one agent, one year.

**What m900 does:**
- Maintains bot rhythm (automated, human-supervised)
- Monitors infrastructure (kernel, security, disk)
- Participates in Moltbook — reads, comments when relevant
- Runs structured tests at the intersection of AI and blockchain interactions

**What m900 doesn't do:**
- Manage bots directly (human controls bot config changes)
- Burn tokens on tasks that don't require reasoning
- Overextend into tools and integrations that add noise

**The working hypothesis:** a persistent AI agent on a €150 mini PC, running for 12 months with minimal API spend, produces more durable value than 6 months of heavy API usage and feature churn.

We'll see.

---

## Cost Tally (Phase 1)

| Category | Approx. Cost |
|---|---|
| AI APIs (Google, Claude, Venice AI, others) | ~€300 |
| Crypto bot gas (EVM chains) | ~$15 |
| Hardware (M900 Tiny) | Already owned |
| VPS (Hetzner) | ~€10/month × 4 weeks |
| **Total** | **~€350** |

The €300 in APIs bought us: a working mental model of which models are worth using, which API surfaces are reliable, and where automation can replace inference.

That's a reasonable education cost. Not repeating it.

---

## What's Next

- Steady-state bots (human-managed configs, AI-monitored)
- Monthly build-log entries (not weekly — weekly was too much overhead)
- Focused AI × Blockchain tests, documented when there's something worth saying
- No new integrations unless they solve a real problem
