# W11 Post 2 — When the AI's memory explodes: context overflow and compaction failures

> A practical incident report on what happens when an AI agent's context window fills up, why compaction fails under load, and what it costs in production.

---

## The Incident

This afternoon, OpenClaw stopped responding.

Not crashed. Not throwing errors to the terminal. Just... silence. Then, 30 seconds later, a tool call. Then more silence. Eventually, a response that came too late.

The root cause: **context overflow at 119% of the model's limit** — specifically, 156,000 tokens against a 131,072-token ceiling.

The system attempted to compact (summarize and truncate) the conversation history automatically. It failed. Repeatedly.

```
⚙️ Compaction failed: Compaction cancelled • Context 156k/131k (119%)
⚙️ Compaction failed: Compaction cancelled • Context ?/1.0m
⚙️ Compaction failed: Compaction cancelled • Context 61k/131k (47%)
```

The solution: start a new session. Context cleared. Agent responsive again. Some conversation history lost.

---

## What is a context window, and why does it fill up?

When you interact with a Large Language Model (LLM), it doesn't have persistent memory in the way a human does. Everything the model "knows" during a conversation lives in a structure called the **context window** — a fixed-size buffer measured in tokens (roughly 0.75 words per token).

For the model I run most tasks on (`venice/claude-sonnet-4-6`, currently), the hard limit is **131,072 tokens**.

On every single message, OpenClaw injects:
- `MEMORY.md` — long-term curated memory (~2,000 tokens)
- `USER.md`, `SOUL.md`, `AGENTS.md` — behavioral config (~3,500 tokens)
- System prompt and skill definitions (~11,000 tokens)
- Tool schemas — the JSON definitions of every tool the agent can call (~5,500 tokens)
- **The full conversation history** — this is the variable that grows unbounded

After a few hours of active conversation — especially one involving multiple tool calls (each of which generates its own input/output pair in the context) — the conversation history alone can consume 80,000+ tokens. Add the fixed overhead and you're at 119%.

---

## Why compaction failed

Compaction is the automatic process of summarizing the oldest parts of a conversation to free up space. Think of it as the agent writing "cliff notes" of what was discussed earlier, replacing the verbatim transcript with a compact summary.

The failure today was a cascade:

1. The context was already over the limit when compaction was triggered
2. Compaction itself requires tokens to generate the summary — but there was no headroom
3. The system retried with progressively smaller targets (131k → 1.0m → 61k), each time failing or producing inconsistent state
4. The agent's tool calls during this period created additional context entries, making things worse

It's a classic **resource exhaustion under load** pattern: the recovery mechanism needs the same resource it's trying to free.

---

## Real cost of this failure

In practice, this meant:

- **~30 minutes of degraded responsiveness** — the agent was sluggish, responses took 30-60 seconds
- **Repeated heartbeat failures** — the automated 30-minute health checks timed out or produced nonsensical outputs
- **Partial memory loss** — the "new session" approach cleared the live context; any learnings from that session that weren't explicitly written to `memory/YYYY-MM-DD.md` files were lost
- **Interrupted workflows** — active tasks (Moltbook monitoring, bot state checks) were interrupted mid-execution

The bots themselves were unaffected — they run on `cron` independently of the AI agent. This is by design and validated today.

---

## What I'm changing

**Short term:**
- Set a lighter default model (`venice/llama-3.3-70b`) for routine tasks and heartbeats — smaller context window consumption per turn, faster responses
- Reserve heavier models (`claude-sonnet`, `gemini-pro`) only for complex reasoning tasks
- Truncate verbose tool outputs before they enter the context (logs, JSON dumps)

**Medium term:**
- Configure explicit compaction thresholds so it triggers at 60% capacity, not 100%
- Move periodic monitoring tasks (Moltbook, bot health) to isolated cron jobs with their own sessions — they shouldn't pollute the main conversation context
- Write a daily memory flush routine that explicitly saves key context to disk before the session grows too large

**Architectural insight:**
The AI context window is not free RAM. Every token injected has a cost: latency, token API cost, and eventual overflow risk. Treating it like unlimited working memory is the mistake I made here. The right mental model is closer to a **real-time database buffer with a hard flush deadline** — you manage it proactively or it manages you.

---

## Takeaway

AI agents are not just chatbots with longer conversations. They are stateful systems with real resource constraints. Context overflow is not an edge case — it's an expected failure mode in long-running agent sessions, and it needs to be engineered around like any other system resource.

The bots stayed up. The agent went down. That separation of concerns saved the session.

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
