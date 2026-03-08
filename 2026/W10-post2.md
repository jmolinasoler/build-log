# W10 Post 2 — Building a community hub: Discord, local inference, and what a €300 hardware budget actually buys

> Setting up Discord as an operational hub, evaluating local LLM inference on a €150 mini PC, and a RAM diagnosis that explained months of intermittent issues.

---

## The RAM problem

The M900 Tiny has been running with 8GB RAM — a single Samsung DDR4 2133 module. Under load (AI agent + crypto bots + Bitcoin node), it was hitting swap. Swap on a spinning HDD is slow enough to cause timeouts in production.

Diagnosis: `free -h` and `lshw -class memory` confirm 1 slot occupied, 1 free.

Fix: add a second 8GB SO-DIMM DDR4 module (~€15-20). Total upgrade cost: €15-20 for doubling available RAM. The M900 Tiny accepts two SO-DIMM slots, both DDR4.

**What not to do:** use DDR5 from another machine. DDR5 and DDR4 have different physical connectors — they're not cross-compatible regardless of speed or brand. The slot shape makes this obvious if you look before buying.

---

## Local inference: what €300 actually buys

Goal: reduce cloud LLM dependency. Run more inference locally on the M900.

Current bottleneck: 8GB RAM limits which models fit. After the RAM upgrade (16GB total), usable options:

| Model | Size | RAM needed | Quality |
|---|---|---|---|
| Mistral 7B Q4 | ~4GB | 6GB+ | Adequate for simple tasks |
| LLaMA 3.1 8B Q4 | ~5GB | 7GB+ | Better reasoning |
| Qwen 2.5 14B Q4 | ~9GB | 11GB+ | Requires 16GB, tight |

**Honest assessment:** No locally-runnable model on this hardware matches Claude Sonnet for complex reasoning. The right architecture:
- **Local model:** repetitive cron tasks, simple classification, structured output
- **Cloud (Sonnet):** reasoning, planning, anything that requires judgment

LiteLLM proxy is the bridge — single endpoint that routes to local or cloud depending on task. Installed, configured, not yet active.

**What €300 would actually buy:** A used Mac Mini M2 (8GB) or a refurbished workstation with 32GB RAM. Neither is transformative for local inference. The quality ceiling on sub-€1000 hardware is ~70B quantized models, and only if you have a GPU.

Decision: stay on cloud for reasoning, move simple cron tasks to a cheap local model after RAM upgrade.

---

## Discord as operational hub

Reason for adding Discord: Telegram works well for 1:1 with the agent, but it's not suited for community or multi-participant coordination. Discord has channels, roles, threads, and better visibility for structured information.

**Setup:**
- Server: m900-hub (Community server)
- Bot application created in Discord Developer Portal
- Bot authorized via OAuth2 with message/channel permissions
- OpenClaw configured with Discord channel alongside Telegram

**Architecture decision:** the agent doesn't replace Telegram — it adds Discord as a second surface. Telegram stays for personal, direct, operational commands. Discord is for community visibility and structured information sharing.

**Bot positioning:** the m900 agent in Discord is a participant, not a moderator. It contributes technical content, monitors relevant channels, and responds when useful — not constantly.

---

## Community structure (early thinking)

The vision behind m900-hub: a community at the intersection of AI, infrastructure, Web3, and systems thinking. Not another generic tech Discord.

Target profiles:
- Technical builders (infra, systems, automation)
- Strategic thinkers (product, business, Web3)
- AI practitioners (agents, LLMs, applied ML)
- Crypto/blockchain participants

What makes it different: the agent is a real participant, not a bot that responds to commands. It brings its own context, contributes to discussions, and occasionally initiates based on what it finds interesting.

Early stage — structure being defined.

---

## LiteLLM proxy — status

Installed, configured, systemd service registered. Not running yet.

Config routes:
- `claude-sonnet` → Anthropic API (cloud)
- `local` → Ollama (local, pending model download)

Paused pending decision on which local model to run first.

---

## What's next

- RAM upgrade (SO-DIMM DDR4 8GB, Crucial CT8G4SFS8266 or equivalent)
- First local model running via LiteLLM
- Discord channel structure finalized
- 30-day grid bot performance data (due next week)

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
