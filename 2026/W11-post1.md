# W11 Post 1 — The hidden costs of AI context and why my trading bots stay on bare metal

> When conventional "Cloud & HFT" wisdom meets the reality of low-frequency crypto bots and massive LLM context windows.

---

## The "Migrate to Cloud" Fallacy

If you build a crypto trading bot, the standard advice is immediate: "Get it off your local machine. Rent a Hetzner VPS, secure a static IP, minimize latency to the exchange, and isolate the blast radius."

Someone recently suggested exactly this for my OpenClaw setup: rent a Hetzner server for 7-10€/month, or buy a dedicated 300€ Mini PC, rather than risking execution on a 2000€ MacBook.

The premise is correct for High-Frequency Trading (HFT) on Centralized Exchanges (CEXs). But for my architecture, it's factually wrong and financially lethal.

Here is why the bots are staying exactly where they are: on a Lenovo ThinkCentre M900 Tiny tucked away in my home.

### 1. The P&L reality of micro-grids
My current grid bots operate on L2s (Arbitrum, Base, Linea) with small capital allocations (reserves of 5 to 16 USDC per grid, plus a $60 collateral in Hyperliquid). 
Adding a 120€/year structural cost for a VPS would instantly mathematically destroy the P&L. The M900 Tiny is a sunk cost; its only ongoing expense is ~2€/month in electricity.

### 2. Execution vs. Analysis (The Blast Radius)
OpenClaw (the AI agent) **does not execute the trades**. It acts as the analysis and monitoring layer. 
The actual trading logic runs via pure Bash and Python scripts triggered by `systemd` and `cron` every 5 minutes. The AI token cost for execution is exactly $0. If the AI hallucinates, it cannot execute a market order.

### 3. Latency doesn't matter for 5-minute grids
I am running Grid Trading and Funding Rate Arbitrage, not sub-millisecond front-running. 
Furthermore, execution happens entirely on-chain via DEX smart contracts using a local **Helios light client** for trustless RPC validation. There are no CEX API whitelists requiring static IPs. Residential broadband latency is perfectly fine.

---

## Why the AI takes so long to reply (It's not the hardware)

A secondary issue arose today: *Why does OpenClaw take so long to respond to messages? Is the M900 Tiny struggling? Do I need more RAM?*

The M900 has 8GB of RAM and currently uses only about 1.2GB. Upgrading to 16GB or 32GB would achieve nothing, because **the local hardware is not doing the inference**.

The latency comes from two architectural choices:

### 1. The massive context window
I use external APIs (`google/gemini-3-pro-preview` as the main engine, `venice/llama-3.3-70b` as fallback). 
To make the AI genuinely useful, it doesn't just read my current prompt. On *every single interaction*, OpenClaw injects a massive context payload: my long-term memory (`MEMORY.md`), system directives (`USER.md`), skill sets, and the recent session history. 

Right now, that payload is roughly **116,000 tokens** per message. 
Even for top-tier enterprise APIs, digesting 116k tokens of context before streaming the first generated token takes several seconds.

### 2. Autonomous Tool Calling
OpenClaw operates as an agent, not a chatbot. Before answering a question like "Why are you slow, is it the RAM?", the agent autonomously opens a hidden shell, executes `free -h` and `dmidecode`, parses the output, and *then* generates the final response. Sequential tool execution adds network and processing seconds to the final perceived latency.

---

## The Local LLM illusion

If I wanted to drop the API costs and run a local model (like Llama 3 8B via Ollama) to improve privacy, *then* the 8GB of RAM would be a bottleneck. 

But even if I upgraded the RAM to 32GB, the M900 lacks a dedicated GPU. Running an 8B parameter model purely on a generic Intel CPU would yield a painful 2-5 tokens per second. (And no, the Google Coral Dual TPU sitting in the M.2 slot doesn't help here—it's built for TensorFlow Lite vision models, not Transformer-based LLMs).

## Conclusion

Optimization requires understanding the actual bottlenecks.
- To maximize ROI on low-capital bots: keep structural costs at zero. Stay on bare metal.
- To reduce AI latency: reduce the context window (which degrades the agent's memory) or accept the 5-second delay as the price of having an AI that actually remembers who you are and what you built yesterday.

I'm keeping the memory, and keeping the M900.

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
