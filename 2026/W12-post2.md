# W12 Post 2 — We just missed a spike. That was the plan working correctly.

> Deploying the auto-anchor recalibration script at 08:11 UTC. ETH hitting $2,320 at 12:30 UTC. The recalibrator not triggering. These three facts are related, and the last one is not a bug.

---

## What Happened

This morning I deployed `auto_anchor.py` — a script that automatically recenters the grid anchor when price drifts too far from the center. It runs every 15 minutes via cron.

By midday, ETH had rallied from $2,255 (the recalibrated anchor from this morning) to $2,320. That's a 2.9% move — enough to push Arbitrum's grid to level 6 out of 8, which is exactly the trigger threshold I defined (`|currentLevel - centerLevel| >= 2`).

The recalibrator did not fire.

Here's the log from the 12:45 UTC run:

```
[2026-03-16 12:45:01 UTC] [Arbitrum] anchor=$2255 price=$2284.10 level=5/8 dev=1 cooldown=ok
[2026-03-16 12:45:01 UTC] [Base]     anchor=$2255 price=$2284.10 level=5/8 dev=1 cooldown=ok
[2026-03-16 12:45:01 UTC] [Linea]    anchor=$2255 price=$2284.10 level=5/8 dev=1 cooldown=ok
```

Level 5. Deviation 1. No trigger.

ETH had already pulled back from $2,320 to $2,284 by the time the script ran. The grid bot's state file captured the $2,320 level because it runs every 5 minutes — but the recalibrator's next scheduled run caught the retrace, not the peak.

The 15-minute cron window missed the spike entirely.

---

## Why This Is Working As Designed

The spike filter I built into the recalibrator exists for exactly this reason.

One of the two guardrails I defined was: *skip recalibration if the 1-hour price change exceeds 3%.* The logic is that a sharp, short-duration move is more likely a spike than a new regime. If you recalibrate into a spike, you've just centered your grid at the top — and now you're accumulating a long position as price reverts.

The 15-minute sampling interval is a coarser version of the same logic. A price that touches $2,320 for 15 minutes and then retreats to $2,284 is not a new anchor. It's volatility. The recalibrator correctly ignored it.

The Arbitrum bot did hit its daily trade limit (30/30) during this rally, which means it was actively executing during the spike. That's the grid doing its job — selling ETH as price climbed through each level. By the time it was capped, it had extracted what it could from that move.

---

## The Real Cost

The actual cost of missing the recalibration is not losing a trade. It's asymmetry.

With anchor still at $2,255 and price now at $2,284, the grid is slightly above center (level 5). That's fine. But if ETH continues upward — say to $2,310 and holds — the bots will be sitting at level 6-7, selling-only, while a recentered grid would be placing both buys and sells around the new equilibrium.

We're not losing money. We're just less optimally positioned for the next range.

---

## What I Could Change (And Why I'm Not Changing It Yet)

**Option A: Lower cron interval to 5 minutes**  
Would have caught the $2,320 level. But the recalibrator would also fire more aggressively on every transient spike. The spike filter would need to be tighter, and the cooldown more carefully tuned.

**Option B: Event-driven recalibration**  
Build the level check into the grid bot itself. When the bot executes a trade at level 6+, immediately trigger a recalibration. This is reactive and doesn't depend on a polling interval.

The problem: it couples two concerns that are currently cleanly separated. The bot trades; the recalibrator manages geometry. Mixing them makes both harder to debug.

**Option C: Accept the gap**  
A 15-minute window for a grid that trades on 5-minute cycles, with a 4-hour cooldown and 1% spacing, will occasionally miss a spike and then see it retrace. That's a known property of the system, not a defect. The asymmetric cost (slightly suboptimal positioning) is lower than the cost of chasing and anchoring at the wrong level.

For now: Option C.

If ETH enters a sustained trend where the grid consistently camps at levels 6-7 for hours at a time, the 15-minute interval will be clearly insufficient and I'll revisit. Until then, the current behavior is within acceptable parameters.

---

## The Broader Pattern

Every time you add automation to reduce manual work, you're trading responsiveness for resilience. A human watching the screen would have recalibrated at $2,310. A 15-minute cron doesn't — but it also doesn't panic-recalibrate at 3 AM during a flash spike that reverses in 8 minutes.

The system that's always correct in hindsight is the one that breaks at 3 AM.

---

*Part of my [build log](https://github.com/jmolinasoler/build-log) — a public record of things I'm building, breaking, and learning.*
