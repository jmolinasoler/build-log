# Week 1 — March 2026

> First entries. Setting up the foundation: local AI agent, Edge TPU, automated trading infrastructure.

---

## Google Coral Edge TPU — Working on Python 3.12

**Problem:** Google's official `pycoral` library has no wheel for Python 3.12. Every install guide tells you to use `pycoral` — it silently installs the wrong package (a reef mapping tool with the same name on PyPI).

**What I did:**
1. Compiled the `gasket`/`apex` DKMS driver for kernel 6.8 (required 2 patches — upstream driver doesn't build cleanly on 6.8)
2. Dropped `pycoral`. Used `ai-edge-litert` instead — Google's 2024 successor to `tflite-runtime`
3. Loaded `libedgetpu.so.1` directly as a delegate

```python
from ai_edge_litert import interpreter as lit
delegate = lit.load_delegate('libedgetpu.so.1')
```

**Result:** Edge TPU delegate confirmed working. Hardware: Google Coral USB Accelerator on Lenovo ThinkCentre M900 Tiny (Ubuntu 24.04, kernel 6.8.0-101).

**Key lesson:** Google's own documentation is outdated. `ai-edge-litert` is the correct package as of 2024. No one had written this up for Python 3.12 + Ubuntu 24.04 at the time I did it.

→ Full write-up: [projects/coral-edge-tpu.md](../projects/coral-edge-tpu.md)

---

## Local AI Agent Infrastructure

Running [OpenClaw](https://github.com/openclaw/openclaw) on the M900 Tiny as a persistent AI agent with:
- Persistent memory across sessions (daily notes + long-term MEMORY.md)
- Telegram integration (proactive alerts, commands, responses)
- Cron-based automation (crypto price alerts, system health checks, trading bots)
- All inference local or via API — no cloud dependency for orchestration

**Why this matters:** The agent manages real infrastructure autonomously. It evaluates bot performance with data, recalibrates parameters, and sends alerts. Not a demo — production.

---

## Algorithmic Grid Trading — Data-Driven Evaluation

Built and evaluated a memecoin trading bot on Solana. Ran it for 2 weeks on real capital.

**Results:**
- Win rate: 14.3%
- Expectancy: −$18.2 per trade
- Total P&L: −$56.53

**Decision:** Shut it down. Pivoted to grid trading (SOL/USDC, ETH/USDC across Arbitrum, Base, Linea).

**Why grid over momentum:** Bear/sideways market conditions. Grid bots profit from oscillation; momentum bots need trend. Reading the market, not fighting it.

Current setup: 4 bots in production, running on system cron every 5 minutes. Zero cloud cost.

→ Full write-up: [projects/grid-bots.md](../projects/grid-bots.md)

---

## What's Next

- Deep-dive write-up on Coral Edge TPU setup (the one that would have saved me 4 hours)
- Evaluating grid bot performance at 30-day mark
- MiCA exam prep (March 9, 2026) — crypto regulatory framework
