# How to Cap LLM Spend on Autonomous OpenClaw Agents

> _A human agent stops spending when they go to sleep. An autonomous one doesn't. The whole point is that it runs without you — which means a runaway loop spends without you, too._

---

## The problem with "set it and forget it"

Autonomous agents — heartbeats, cron jobs, always-on assistants — are the best reason to run OpenClaw and the scariest thing to put a wallet behind. A misfiring loop, a context that won't stop growing, or a provider that keeps returning retriable errors can all turn into spend that nobody is watching in real time.

OpenClaw users have been asking for a spending ceiling for a long time. From [#42475](https://github.com/openclaw/openclaw/issues/42475) (11 comments):

> _"Per-agent cost budget enforcement at the gateway level."_

From [#17683](https://github.com/openclaw/openclaw/issues/17683):

> _"Scoped / Script-Limited Agent Mode (Cost-Controlled + Predictable Execution)."_

From [#64463](https://github.com/openclaw/openclaw/issues/64463):

> _"session.maxDurationMinutes and session.maxTokensPerSession config keys."_

The common thread: people want a hard limit that the agent cannot exceed, enforced by the infrastructure rather than by the agent's own good behavior. An autonomous agent policing its own budget is a fox guarding a henhouse.

And you can't enforce a budget you can't see. From [#13219](https://github.com/openclaw/openclaw/issues/13219) and [#57404](https://github.com/openclaw/openclaw/issues/57404), users are still asking OpenClaw for per-model usage logging and per-run token usage on the event stream — the visibility that any budget has to be built on.

---

## How ClawRouter solves this

ClawRouter sits between OpenClaw and the providers, which makes it the right place to enforce a spend ceiling: it sees the real, post-routing cost of every call and can act _before_ a request is sent. The controls are wallet-native — there's no subscription to overshoot and no surprise invoice, because the agent literally cannot spend USDC it doesn't have, and you decide how much of that it's allowed to spend per run.

### A hard per-session cost cap

ClawRouter enforces a per-run (per-session) dollar ceiling via `maxCostPerRunUsd`. This is the enforced budget [#42475](https://github.com/openclaw/openclaw/issues/42475) is asking for, applied at the router:

```jsonc
// ~/.openclaw/openclaw.json — illustrative
{
  "maxCostPerRunUsd": 0.5, // this session may spend at most $0.50
  "maxCostPerRunMode": "graceful", // how to behave as it approaches the cap
}
```

ClawRouter tracks accumulated session cost as the run proceeds and checks each request's projected cost against the remaining budget before it goes out. [#64463](https://github.com/openclaw/openclaw/issues/64463) frames the limit in tokens; ClawRouter frames it in dollars — the same ceiling, expressed in the unit you actually care about on the invoice.

### Two enforcement modes: degrade or stop

The cap has two behaviors, because "cap reached" shouldn't always mean "work stops":

- **`graceful` (default)** — as the session nears its budget, ClawRouter **downgrades to cheaper models** and falls back to a **free model** as a last resort. The agent keeps working; it just gets cheaper as it approaches the line. It only hard-stops if no model can serve the request within budget.
- **`strict`** — the moment session spend reaches the cap, ClawRouter returns a `429` and stops. This is the "predictable, script-limited" behavior [#17683](https://github.com/openclaw/openclaw/issues/17683) describes: a run that physically cannot exceed its number.

```
[ClawRouter] session spend $0.46 of $0.50 cap — graceful: downgrading to free tier
[ClawRouter] served by free/gpt-oss-120b ($0.00)
```

Pick `graceful` for assistants you want to stay useful, `strict` for cron jobs you want to stay bounded.

### A $0 floor for the work that doesn't need to cost anything

Graceful mode leans on a free tier that's always available: NVIDIA-hosted models, no balance required, including a 1M-context DeepSeek V4 and a vision-capable Nemotron Omni. For an autonomous agent doing routine polling, classification, or summarization, you can run the whole lane at zero cost — and the cap only ever bites on the genuinely expensive turns.

You can also use `blockrun/free` as the agent's profile outright, which makes the budget question moot: a free-tier agent has a spend ceiling of $0 by construction.

### Ban the models you never want a loop to reach

A runaway loop is most dangerous when it can reach your most expensive model. `/exclude` removes specific models from routing entirely, so even a misbehaving agent can't escalate into them:

```bash
/exclude anthropic/claude-opus-4.8   # never let an autonomous loop reach Opus
```

Excluded models are filtered out of every fallback chain, not just the primary — so there's no back door.

### Spend you can actually see

`/stats` reports what ClawRouter spent and which models served the requests — the per-model breakdown [#13219](https://github.com/openclaw/openclaw/issues/13219) is requesting. Combined with the wallet's own balance (ClawRouter warns as it runs low), you always know what an autonomous agent has spent without waiting for a monthly bill.

---

## A recommended setup for an unattended agent

For a cron job or heartbeat you won't be watching:

```jsonc
{
  "agents": {
    "watcher": {
      "model": "blockrun/auto",
      "maxCostPerRunUsd": 0.25,
      "maxCostPerRunMode": "strict", // bounded: cannot exceed $0.25 per run
    },
  },
}
```

Then:

1. `/exclude` your most expensive models so no loop can reach them.
2. Fund the wallet with **only what you're willing to lose** — the wallet balance is itself a hard ceiling. Treat it as a spending account, not a vault.
3. Check `/stats` after the first day and tune the cap to the real cost.

The principle is simple: don't trust an autonomous agent to limit its own spend. Put the limit where the money flows — at the router and the wallet — so the ceiling holds even when the agent misbehaves.

> **A note on what's enforced today.** ClawRouter's runtime ceiling is the per-session `maxCostPerRunUsd` cap described above, plus the wallet balance itself and `/exclude`. Finer-grained rolling windows (hourly/daily limits) are on the roadmap — for now, the per-session cap and a deliberately small wallet top-up are the dependable guardrails for unattended agents.

---

## Related documentation

- [Why Your OpenClaw Bill Spiked 10x](./openclaw-token-cost-spike-background-tasks.md) — where unattended cost actually comes from
- [Routing Profiles](./routing-profiles.md) — `auto` / `eco` / `free` and how downgrade works
- [11 Free AI Models, Zero Cost](./11-free-ai-models-zero-cost-blockrun.md) — the $0 floor
- [Configuration](./configuration.md) — full config reference
- [Report a problem or request a feature](https://github.com/BlockRunAI/ClawRouter/discussions)
