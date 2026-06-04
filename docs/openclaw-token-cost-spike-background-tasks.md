# Why Your OpenClaw Bill Spiked 10x (And You Didn't Change a Thing)

> _You didn't add agents. You didn't change models. But your token usage tripled overnight. The cost isn't in the prompts you wrote — it's in the work the agent does **between** them._

---

## The pattern nobody warns you about

You set up OpenClaw, picked a model, and things were fine. Then one week the bill jumped — sometimes 5x, sometimes 10x — with no change to how you were using it. The OpenClaw issue tracker is full of this exact story, and the root cause is almost never the model you chose. It's the **invisible internal work**: context compaction, heartbeat replays, memory search, and summarization that run on every turn and quietly burn tokens on whatever model happens to be configured.

From [#90170](https://github.com/openclaw/openclaw/issues/90170):

> _"Possible token/cost regression after OpenClaw v2026.5.28 with DeepSeek v4-pro."_

A version bump, not a usage change, moved the needle. That's the tell: the cost lives in machinery you don't directly control.

---

## Where the tokens actually go

Four internal mechanisms dominate the runaway-cost reports. None of them are prompts you typed.

### 1. Compaction firing too often, on the wrong model

When a conversation grows, OpenClaw compacts history into a summary. If that trigger is miscalibrated, it fires repeatedly — each compaction is a full model call over your entire context.

From [#72964](https://github.com/openclaw/openclaw/issues/72964):

> _"Compaction emits empty fallback summary; tokensBefore counts cacheRead, triggering premature compactions on Opus 1M."_

From [#48579](https://github.com/openclaw/openclaw/issues/48579):

> _"Context pruning mode 'off' not preventing compactions."_

And [#81856](https://github.com/openclaw/openclaw/issues/81856) asks for an _"absolute-token trigger for compaction (independent of model context window)"_ — because on a 1M-context model, the percentage-based trigger lets context balloon before it fires, making each compaction enormous.

The damage compounds: premature compaction on a 1M-context flagship means you're paying flagship per-token rates to summarize hundreds of thousands of tokens, over and over.

### 2. Heartbeat replaying context on every beat

Autonomous setups run a heartbeat. If each beat replays prior context, cost grows on a loop with no human in it.

From [#84218](https://github.com/openclaw/openclaw/issues/84218):

> _"Heartbeat isolatedSession=true replays prior heartbeat context, causing deterministic overflow/restart loop."_

From [#65161](https://github.com/openclaw/openclaw/issues/65161) (14 comments):

> _"Heartbeat isolated mode: … lightContext stays heavy …"_

A "light" context that stays heavy means every idle beat costs as much as an active one.

### 3. Memory and summarization lanes pinned to a premium model

Active-memory search, taste-learning, and summarization each call a model. When those lanes inherit your premium default, every background lookup is billed at flagship rates — invisibly.

### 4. You can't even see it

From [#57404](https://github.com/openclaw/openclaw/issues/57404):

> _"Expose per-run token usage on the WebSocket lifecycle event stream."_

From [#13219](https://github.com/openclaw/openclaw/issues/13219):

> _"Per-model usage logging for cost tracking."_

You can't optimize what you can't measure. Both of these are open requests because OpenClaw doesn't, by default, show you which lane spent what.

---

## How ClawRouter solves this

ClawRouter is a local router that sits between OpenClaw and the model providers. Every request — including the internal ones — goes through `blockrun/auto`, which classifies the call across 15 dimensions in under a millisecond and routes it to the cheapest model that can actually handle it. The fix for runaway background cost has three parts.

### Fix 1 — Route the expensive internal lanes to cheap (or free) models

This is the single biggest lever. Point compaction, memory, and summarization at a cost-optimized ClawRouter profile instead of your premium default:

```jsonc
// ~/.openclaw/openclaw.json — illustrative
{
  "agents": {
    "main": {
      "model": "blockrun/auto", // your real work: smart-routed
      "compaction": { "model": "blockrun/eco" }, // summaries don't need a flagship
      "memory": { "model": "blockrun/free" }, // lookups can be free
    },
  },
}
```

ClawRouter ships four routing profiles, so you can dial each lane to the right cost tier:

| Profile            | What it does                                   | Best for                          |
| ------------------ | ---------------------------------------------- | --------------------------------- |
| `blockrun/auto`    | Balanced — cheapest **capable** model per call | Your primary agent work           |
| `blockrun/eco`     | Cheapest possible; free models as primaries    | Compaction, summarization         |
| `blockrun/free`    | 100% free NVIDIA-hosted models, no balance     | Memory lookups, classification    |
| `blockrun/premium` | Best quality regardless of price               | The rare turn that truly needs it |

The free tier matters here: it includes NVIDIA-hosted models with up to **1M context** (DeepSeek V4) and a vision-capable Nemotron Omni — more than enough to summarize a conversation or run a memory lookup at **zero cost**. A compaction call that was billed at flagship rates becomes free, and it runs hundreds of times a day.

### Fix 2 — Compress what does get sent

ClawRouter applies request- and response-side compression before a paid call ever leaves your machine. Verbose tool outputs — the kind that pile up in agentic loops and then get replayed through compaction — are summarized aggressively, and repeated responses are served from a short-TTL cache instead of re-billed. Our cost teardown ([ClawRouter Cuts LLM API Costs 500x](./clawrouter-cuts-llm-api-costs-500x.md)) walks through the layered compression in detail; the practical effect is that the tokens feeding your compaction trigger are smaller, so it fires less and costs less when it does.

### Fix 3 — A hard ceiling per run

Premature compaction loops and heartbeat replays are exactly the failure mode that quietly drains a wallet. ClawRouter enforces a per-run cost cap with two modes:

- **`graceful` (default)** — as a session approaches its budget, ClawRouter downgrades to cheaper models and falls back to a free model as a last resort, so work continues at lower cost instead of stopping.
- **`strict`** — once the session reaches the cap, ClawRouter hard-stops with a `429` rather than letting a runaway loop spend unbounded.

```jsonc
{
  "maxCostPerRunUsd": 0.5,
  "maxCostPerRunMode": "graceful",
}
```

That turns "my bill spiked 10x overnight" into "my background lane downgraded itself and capped the run."

### Visibility, finally

`/stats` reports what ClawRouter actually spent and which models served your requests — the per-model breakdown that [#13219](https://github.com/openclaw/openclaw/issues/13219) and [#57404](https://github.com/openclaw/openclaw/issues/57404) are asking OpenClaw for. When the bill moves, you can see _which lane_ moved it.

---

## A 60-second audit

If your OpenClaw cost jumped without a usage change, check these in order:

1. **What model is compaction on?** If it's your premium default, point it at `blockrun/eco`. This alone often halves a runaway bill.
2. **What model does memory/summarization use?** Send it to `blockrun/free`.
3. **Is a heartbeat replaying context?** Cap it with `maxCostPerRunUsd` in `strict` mode so a replay loop can't run unbounded.
4. **Can you see per-lane spend?** Run `/stats` and watch which model dominates the call count — that's your culprit.

The lesson from the issue tracker is consistent: the prompts you write are rarely the problem. The work the agent does between them is. Route that work to the right cost tier and the spike disappears.

---

## Related documentation

- [ClawRouter Cuts LLM API Costs 500x](./clawrouter-cuts-llm-api-costs-500x.md) — the layered token-compression teardown
- [Routing Profiles](./routing-profiles.md) — `auto` / `eco` / `premium` / `free` reference
- [11 Free AI Models, Zero Cost](./11-free-ai-models-zero-cost-blockrun.md) — what the free tier covers
- [We Read 100 OpenClaw Issues About OpenRouter](./clawrouter-vs-openrouter-llm-routing-comparison.md) — the structural case for local routing
- [Report a problem or share your setup](https://github.com/BlockRunAI/ClawRouter/discussions)
