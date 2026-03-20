# W12 Post 6 — Tuning AI compaction: from "it crashed" to "it triggers early"

> After the context overflow incident, I went into the config and tried to make the compaction system behave. Here's what I found, what broke, and what actually worked.

---

## Context

Last week I documented a context overflow incident where OpenClaw's compaction failed repeatedly at 119% capacity.

In the post I listed three things I'd change. This entry is about actually doing one of them: **configuring the compaction system to trigger earlier, not after it's already too late.**

---

## The exploration

Started by checking the maintenance logs to see what the agent had been doing:

```bash
less /home/m900/.openclaw/logs/maintenance.log
```

Then tried to set a compaction threshold. First attempt:

```bash
openclaw config set agents.main.compaction.min_threshold 0.1
openclaw config set agents.defaults.compaction.min_threshold 0.1
```

Neither worked as expected. Did a config get to inspect the actual structure:

```bash
openclaw config get agents
```

That's when I found the issue: the key changed between versions. In v2026.3, it's `threshold`, not `min_threshold`.

```bash
openclaw config set agents.defaults.compaction.threshold 0.1
```

---

## Trying different modes

With the threshold set, I also experimented with the compaction mode:

```bash
openclaw config set agents.defaults.compaction.mode "summarize"
```

`summarize` forces the agent to always compress when it hits the threshold, regardless of conversation state. In practice this was too aggressive — it was summarizing before I'd finished a task.

Reverted to default:

```bash
openclaw config set agents.defaults.compaction.mode "default"
```

Default mode is smarter: it uses the threshold as a hint, not a hard trigger. Better for interactive sessions.

---

## The two settings that actually mattered

After the back and forth, the two config changes that made a real difference:

**1. `maxHistoryShare`** — limits how much of the context window can be consumed by conversation history before compaction is considered:

```bash
openclaw config set agents.defaults.maxHistoryShare 0.1
```

Setting this to `0.1` (10%) is aggressive, but it means the system is always reserving 90% of the window budget for system prompt, tools, and working context. Compaction is triggered much earlier.

**2. `reserveTokens`** — explicit headroom reservation for the compaction process itself:

```bash
openclaw config set agents.defaults.compaction.reserveTokens 900000
```

This is the fix for the root cause of last week's failure. Compaction failed because it had no tokens left to generate the summary. Reserving 900k tokens means the summarizer always has room to work, even when the rest of the context is full.

Both followed by a gateway restart:

```bash
openclaw gateway restart
```

---

## What changed in behavior

Before: compaction triggered reactively, at ~90-100% capacity, sometimes failing because there was no headroom left.

After: compaction triggers early (when history hits 10% of the window), with a guaranteed token reserve for the summarization process. The agent stays responsive and doesn't enter the degraded state I saw last week.

Downside: compaction runs more frequently. Each compaction is a short model call with a small cost. For now, that's the right tradeoff — predictable behavior beats rare but catastrophic failures.

---

## Lessons

- **API config keys change between versions.** Always `config get` before assuming a key name is stable. `min_threshold` → `threshold` was a silent breaking change between versions.
- **Reserve resources for your recovery mechanism.** If your fallback process needs the same resource it's trying to free, you need to pre-allocate. `reserveTokens` is exactly this.
- **"Default" mode isn't a cop-out.** It genuinely behaves better than forced summarization for interactive sessions. The threshold still applies — you just let the system decide the exact moment.

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
