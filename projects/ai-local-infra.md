# AI-Assisted Local Infrastructure

> Status: 🔄 Ongoing  
> Hardware: Lenovo ThinkCentre M900 Tiny  
> Agent: OpenClaw (self-hosted AI agent framework)  

---

## What I Built

A persistent AI agent running locally that manages infrastructure, monitors systems, and handles real-world automation — not a chatbot demo.

**Hardware:** Lenovo ThinkCentre M900 Tiny (i5-6500T, 8GB DDR4, 477GB SSD)  
**Cost:** ~€150 refurbished. Runs 24/7 at ~15W.  
**Extras:** Google Coral USB Accelerator (Edge TPU for local ML inference)

---

## What the Agent Does

**Autonomous tasks (no human trigger):**
- Monitors 4 trading bots — alerts on anomalies, recalibrates when needed
- Sends Telegram alerts for price thresholds (crypto assets)
- Checks system health: kernel updates, pending reboots, disk usage
- Runs accumulation bot: DCA into BTC/ETH on Kraken when EUR balance detected
- Monitors airdrop tokens on Binance before auto-converting to BNB

**On-demand (via Telegram):**
- Full natural language interface to the server
- File operations, code execution, web search, bot configuration changes
- Portfolio queries across 4 chains + 2 CEXes

---

## Architecture

```
Telegram ←→ OpenClaw Agent (Claude Sonnet)
                ├── Memory: MEMORY.md + daily notes (persistent across sessions)
                ├── Tools: file read/write, shell exec, web fetch, browser control
                ├── Crons: scheduled tasks (system crontab + OpenClaw cron)
                └── Integrations: Kraken API, Binance API, on-chain (Solana/EVM/HL)
```

**Key design principle:** AI for reasoning and decisions. Scripts for execution. Never pay AI inference cost for something a bash script can do.

---

## Why This Matters (for my CV)

This isn't an AI toy. It's operational infrastructure:
- Real money managed autonomously
- Real alerts that prevent losses
- Real decisions made from data (shut down a bot with negative expectancy, not emotion)

The AI layer adds judgment. The script layer adds reliability. The combination is more capable than either alone.

---

## Lessons

1. **Persistent memory is the hard part.** Getting an AI to do something once is easy. Getting it to maintain context across weeks, remember past decisions, and build on them — that requires intentional architecture.

2. **Start with observability.** Before automating decisions, automate observation. Alerts before actions.

3. **Cost discipline.** Every AI inference costs money. Design flows so AI is only invoked when judgment is needed. Routine tasks stay in cron + bash.
