# Your Agent Shouldn't Stall for 14 Minutes When a Provider Goes Down

> _A model provider returns a 503. Your agent doesn't fail over — it freezes. Fourteen minutes later you SIGTERM the gateway by hand. The provider was down for ninety seconds._

---

## The single point of failure you didn't know you had

The moment you pin OpenClaw to one model, you've built a single point of failure. When that provider has a bad minute — a 503, a rate-limit burst, an auth token that won't refresh — there's nothing behind it. The session doesn't degrade gracefully. It stalls.

From [#84865](https://github.com/openclaw/openclaw/issues/84865):

> _"user-switched model has no fallback chain, causing session deadlock on provider outage."_

This is the trap: the act of choosing a model — exactly what a careful user does — strips away the fallback behavior. Your "I'll just use Sonnet" decision quietly disabled the safety net.

It gets worse when the failure isn't a clean error. From [#79611](https://github.com/openclaw/openclaw/issues/79611):

> _"active-memory plugin: pins to one provider, no failover; timeout shorter than provider latency causes silent drops."_

Now the failure is invisible: the provider is just slow, the timeout fires first, and the work silently disappears with no error to react to.

---

## Why naive failover isn't enough

The instinct is "add a fallback list." But OpenClaw users who tried that hit a second wall: **not every failure should be treated the same way.**

From [#47910](https://github.com/openclaw/openclaw/issues/47910) (5 comments):

> _"feat: provider fallback by failure class — quarantine auth-broken providers."_

A 429 (rate limit) is transient — retrying the same provider in a moment often works. A 401 (bad auth) is not — retrying it just wastes ~10 seconds per request before you fail over anyway. If your failover logic can't tell these apart, it either gives up too early on recoverable errors or wastes time hammering a provider that will never answer this session.

And when failures pile up, you want the gateway to stop digging. From [#62615](https://github.com/openclaw/openclaw/issues/62615):

> _"Add gateway-side circuit breaker for unhealthy sessions after consecutive failures."_

Without one, a degraded provider can drag a whole session — or, per [#83366](https://github.com/openclaw/openclaw/issues/83366), the entire gateway event loop — into timeout territory under load.

So real failover needs three properties: **a chain to fall to, the judgment to classify failures, and a floor that's always reachable.** That's what ClawRouter provides.

---

## How ClawRouter solves this

When OpenClaw points at `blockrun/auto`, you're no longer pinned to one model — you're pointed at a router with a fallback chain behind every tier.

### A chain behind every request

ClawRouter classifies each request into a tier (simple / medium / complex / reasoning) and each tier has a **primary plus an ordered list of fallbacks** spanning different providers. When the primary fails, it moves down the chain — and because the fallbacks are on different providers, one provider's outage doesn't poison the others.

```
[ClawRouter] tier=MEDIUM
[ClawRouter] primary moonshot/kimi-k2.6 → 503, falling through chain
[ClawRouter] trying next: google/gemini-3-flash-preview → ok
[ClawRouter] served by google/gemini-3-flash-preview
```

The user-pinned deadlock in [#84865](https://github.com/openclaw/openclaw/issues/84865) can't happen here, because choosing `blockrun/auto` _is_ choosing the chain — the fallback behavior is the default, not something you lose by making a selection.

### Failure classification, not blind retry

ClawRouter classifies upstream failures before deciding what to do, which is exactly the behavior [#47910](https://github.com/openclaw/openclaw/issues/47910) asks OpenClaw for:

- **Auth failures (401/403)** are not retried against the same key — they fall straight through to the next model instead of burning ~10s per attempt.
- **Transient provider errors** are treated as retriable and cascade to the next model in the chain.
- **Payment-simulation failures** (on the x402 settlement path) automatically retry with a different model rather than surfacing a raw error to the agent.

The agent never sees a bare 503 or a raw 401. It sees a result, or — if every model in the chain genuinely fails — a single structured error that lists what was tried.

### A floor that's always reachable

The bottom of every cost-optimized chain is the **free tier**: NVIDIA-hosted models that require no balance and aren't tied to your primary provider's uptime. When a paid provider is mid-outage, ClawRouter can still complete the turn on a free model instead of stalling. The worst case becomes "this turn ran on a free model for a minute," not "the gateway hung until I restarted it."

| Failure mode                    | Pinned single model        | ClawRouter (`blockrun/auto`)             |
| ------------------------------- | -------------------------- | ---------------------------------------- |
| Provider returns 503            | Session stalls / deadlocks | Falls to next model in chain             |
| 401 / auth won't refresh        | ~10s wasted, then error    | Classified, skips straight to next model |
| Provider slow, timeout fires    | Silent drop                | Cascades; free tier as last resort       |
| Every paid provider unavailable | Hard failure               | Completes on a free model                |

---

## What to change today

1. **Stop pinning a single model.** Set your primary to `blockrun/auto` — the fallback chain comes with it.
2. **If you must pin a specific model**, add ClawRouter as the fallback so an outage degrades instead of deadlocking:
   ```bash
   openclaw models set anthropic/claude-sonnet-4.6
   openclaw models fallbacks add blockrun/auto
   ```
   (See [Using Subscriptions with ClawRouter Failover](./subscription-failover.md) for the full setup.)
3. **Watch it work** with `openclaw gateway logs | grep -i "blockrun\|fall"` during your next provider hiccup.

An autonomous agent is only as reliable as its worst provider-minute. The fix isn't a better provider — it's never depending on a single one.

---

## Related documentation

- [Using Subscriptions with ClawRouter Failover](./subscription-failover.md) — keep your subscription primary, ClawRouter as automatic failover
- [We Read 100 OpenClaw Issues About OpenRouter](./clawrouter-vs-openrouter-llm-routing-comparison.md) — broken fallback as the #1 router pain point
- [Routing Profiles](./routing-profiles.md) — how tiers and chains are structured
- [11 Free AI Models, Zero Cost](./11-free-ai-models-zero-cost-blockrun.md) — the free-tier safety net
- [Report a problem or share your setup](https://github.com/BlockRunAI/ClawRouter/discussions)
