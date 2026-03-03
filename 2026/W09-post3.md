# W09 Post 3 — Running a second AI agent on a €4/month VPS

> Deploying a specialized OpenClaw agent on Hetzner for basketball coaching — and the lessons from setting it up.

---

## Why a second agent?

The M900 runs my main AI agent — infrastructure, crypto bots, personal context. I wanted a separate agent focused exclusively on basketball coaching: training planning, match analysis, player tracking, calendar integration.

The decision: keep them isolated. Different purpose, different memory, different Telegram bot. No cross-contamination.

**Hardware choice:** Hetzner CX22 (2 vCPU, 4GB RAM, 40GB SSD, €3.79/month). For a text-only assistant with no local ML inference, there's no reason to buy physical hardware.

---

## The setup

Stack on the VPS:
- Ubuntu 24.04 LTS
- Node.js 22 + OpenClaw 2026.3.2
- Venice AI (free inference) as the LLM provider
- Telegram bot (separate from the main agent)
- iCal integration: club calendar (Twizzit) + family calendar (Google)

The agent reads both calendars and sends a weekly summary every Monday at 18:00.

---

## What broke and why

### 1. Double NAT blocked SSH

The M900 sits behind a home router — normal NAT. But it was also behind a second NAT layer at the network level. SSH connections to the Hetzner VPS were silently dropped before the handshake completed.

Fix: moved the M900 to a different VLAN with direct routing. IP changed, SSH worked immediately.

**Lesson:** `kex_exchange_identification: read: Connection reset by peer` before any authentication attempt = network-level block, not SSH config.

### 2. Venice AI: Admin Key vs Inference Key

Venice has two types of API keys:
- **Admin Key** — account management, listing models. Works on `/models` endpoint.
- **Inference Key** — actually running models. Required for `/chat/completions`.

The `401 Authentication failed` error on all models was caused by using an Admin Key for inference. The `/models` endpoint accepts Admin Keys (which made it look like auth was working), but inference calls silently reject them.

```bash
# This works with Admin Key:
curl https://api.venice.ai/api/v1/models -H "Authorization: Bearer <admin_key>"

# This fails with Admin Key, needs Inference Key:
curl https://api.venice.ai/api/v1/chat/completions -H "Authorization: Bearer <admin_key>"
```

Also: Venice inference keys require the full prefix in the Authorization header:
```
Authorization: Bearer VENICE_INFERENCE_KEY_xxxxx
```
Not just the key suffix.

### 3. OpenClaw stores API keys in its own keystore

OpenClaw doesn't read API keys from the JSON config directly. They live in:
```
~/.openclaw/agents/main/agent/auth-profiles.json
```

Setting `apiKey` in the models provider config in `openclaw.json` does nothing. The correct approach is writing directly to `auth-profiles.json` with the right structure.

---

## Calendar integration

Both calendars are iCal feeds — no OAuth, no API keys:
- **Club calendar:** Twizzit exports a private iCal URL per coach
- **Family calendar:** Google Calendar private iCal URL

A single Python script reads both, expands recurring events, and outputs a clean summary. The agent calls it directly via `exec` when asked about the schedule.

```bash
python3 gcal_reader.py --days 7 --cal basketball
# → upcoming training sessions and matches with location
```

---

## Current architecture

```
Julio (Telegram)
    ├── @Jms900_bot → M900 (Brussels LAN)
    │     Infrastructure, crypto bots, personal context
    │     Model: Claude Sonnet 4.6 (Anthropic)
    │
    └── @basketm900_bot → Hetzner CX22 (Falkenstein)
          Basketball coaching, calendar, player tracking
          Model: Claude Sonnet 4.6 (Venice AI — free)
```

Two agents, two bots, one Telegram account. Each knows its domain.

---

## What's next

- Build out player tracking in the basketball agent (roster, stats, progression)
- Add match analysis workflow (post-game notes → structured insights)
- novacifer.com migration from Google Sites to GitHub Pages (Jekyll, steampunk theme — repo live at [github.com/jmolinasoler/novacifer.com](https://github.com/jmolinasoler/novacifer.com))
