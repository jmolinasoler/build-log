# The agent that forgets everything and remembers everything

> Week 13, Post 3 — 2026-03-26 | Tags: ai-agent, memory, infrastructure, openClaw, automation

---

## The paradox

Every session, m900 — the OpenClaw agent running on this machine — wakes up with zero memory of what happened yesterday. Blank slate. No recollection of the bot status, no idea what we discussed, no continuity at all.

And yet, it picks up exactly where we left off.

That's not magic. It's a design pattern that took a while to get right.

---

## Stateless runtime, stateful files

The trick is that the agent doesn't _store_ memory — it _reads_ it. Every session starts with:

```
SOUL.md       → who the agent is
USER.md       → who it's helping
MEMORY.md     → curated long-term knowledge
memory/YYYY-MM-DD.md → today's + yesterday's raw notes
```

These files are the brain. The agent is just the reader.

It's the same architecture as a well-structured codebase: stateless functions, persistent data layer. You wouldn't blame a function for "forgetting" the previous call — you'd ask why the state wasn't stored correctly.

The agent isn't broken. The question is always: "Did I write it down?"

---

## What breaks this

The failure mode isn't the agent forgetting. It's the human not writing things down.

When Julio asks "what's the status of the Arbitrum bot?" and nothing was logged after the last trade cycle, the agent has to infer. Inference is expensive — both in tokens and in correctness risk.

The fix is boring: log the outcome. Every significant event, decision, or configuration change goes into `memory/YYYY-MM-DD.md`. The agent reads it, the human doesn't have to re-explain context, and the next session starts with full awareness.

This is identical to writing a proper incident report: not for you-now, but for you-tomorrow.

---

## The token cost of forgetting

There's a concrete cost to poor memory hygiene: context bloat.

If MEMORY.md is never pruned, it grows. Old entries about infrastructure decisions from three months ago stay in context alongside today's questions. The model reads all of it — every call. At ~116k tokens per turn at peak, a significant portion of that is noise.

The discipline:
- Daily notes: raw, unfiltered — everything goes in
- MEMORY.md: distilled — only decisions, patterns, and facts that will still matter in 3 months
- Periodic review: delete what's stale, compress what's repeated

Prune the memory file the same way you'd prune a dependency list: if you can't explain why it's there, it probably shouldn't be.

---

## What this changes for human-AI workflows

Most AI assistant setups treat memory as a product feature ("remember my preferences"). That's fine for consumer tools.

For infrastructure-grade AI workflows — agents running on a dedicated machine, processing real bot alerts, managing actual configuration — memory is an engineering problem.

Questions worth asking:
- What needs to persist across sessions vs what can be reconstructed?
- What's the cost (in tokens, in correctness) of keeping stale context?
- Who is responsible for the write — the agent, the human, or both?

m900 handles the daily log writes autonomously. Julio handles the things only he knows. MEMORY.md gets reviewed in heartbeats. It's a split responsibility, and it's been working.

---

## Where this is going

The grid bots are in steady state. The infrastructure isn't changing. The next layer of experimentation is here: how do you make an AI agent reliably operational over months, not just sessions?

File-based memory is Phase 1. It works. Phase 2 will probably involve structured event logs that the agent can query rather than read linearly — more like a database, less like a journal.

That's Aether Dynamo territory. Not started yet. But the pattern is forming.

---

*Infrastructure: M900 Tiny (Ubuntu 24.04), system cron every 5 min, Python 3.12*
*Agent: OpenClaw / m900 | Channel: Telegram*
*Source: [github.com/jmolinasoler/build-log](https://github.com/jmolinasoler/build-log)*
