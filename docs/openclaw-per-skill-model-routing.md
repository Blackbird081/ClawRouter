# Per-Skill & Per-Task Model Routing in OpenClaw

> _Not every task deserves your best model. Summarizing a file and architecting a refactor are different jobs — but most setups send both to the same expensive endpoint, because picking the right model by hand for every call is impossible._

---

## One model for everything is the wrong default

OpenClaw lets you pick a model. Singular. That model then handles your hardest reasoning task and your most trivial one-liner at the same per-token price. Users have been asking for finer control across several dimensions, and the requests keep coming because the one-model default leaves real money and real quality on the table.

From [#43260](https://github.com/openclaw/openclaw/issues/43260) (8 comments):

> _"Support `model` field in SKILL.md frontmatter for per-skill model routing."_

People want a coding skill on a strong model and a formatting skill on a cheap one — per skill, not per account.

From [#80521](https://github.com/openclaw/openclaw/issues/80521):

> _"model picker + drag-to-reorder for primary/fallback model selection."_

They want to _see_ and _order_ their models, not hand-edit JSON.

From [#65557](https://github.com/openclaw/openclaw/issues/65557):

> _"Provider & model selection per session/account with admin-controlled allowlists."_

Teams want to restrict which models a given account can even reach.

And from [#88371](https://github.com/openclaw/openclaw/issues/88371):

> _"Windows QuickStart defaults first chat to paid Anthropic model without credit warning."_

The flip side of "pick a model" is that the default pick can quietly be an expensive one — a brand-new user's first message bills against a premium API with no heads-up.

Underneath all four is one need: **the right model for each task, without micromanaging every call.**

---

## How ClawRouter solves this

ClawRouter approaches per-task routing from two directions at once: it routes automatically when you don't want to think about it, and it gives you explicit per-agent control when you do.

### Automatic per-task routing — the default

Point OpenClaw at `blockrun/auto` and ClawRouter classifies **every individual request** across 15 dimensions in under a millisecond, then routes it to the cheapest model that can actually handle that specific task:

```
"reformat this JSON"        → simple tier  → fast, cheap model
"explain this stack trace"  → medium tier  → mid model
"refactor this module"      → complex tier → flagship-quality model
"prove this invariant"      → reasoning    → reasoning model
```

This is per-task routing without any per-task configuration. The formatting call in [#43260](https://github.com/openclaw/openclaw/issues/43260) automatically lands on a cheap model and the hard coding task automatically lands on a strong one — because the router judges each request on its merits, not on a single global setting.

### Explicit per-agent control when you want it

When you _do_ want to pin specific models — the explicit per-skill behavior [#43260](https://github.com/openclaw/openclaw/issues/43260) and [#80521](https://github.com/openclaw/openclaw/issues/80521) are after — OpenClaw's per-agent config plus ClawRouter profiles give you that, today, in `openclaw.json`:

```jsonc
{
  "agents": {
    "coding": {
      "model": "blockrun/premium", // quality-first
      "fallbacks": ["blockrun/auto", "blockrun/free"],
    },
    "formatter": {
      "model": "blockrun/free", // never pay to reformat
    },
    "researcher": {
      "model": "blockrun/auto",
    },
  },
}
```

Each agent (effectively, each lane of work) gets its own cost/quality tier and its own fallback chain. ClawRouter ships four profiles to assign:

| Profile            | Picks                                   | Use it for                 |
| ------------------ | --------------------------------------- | -------------------------- |
| `blockrun/auto`    | Cheapest **capable** model, per request | General-purpose default    |
| `blockrun/eco`     | Cheapest possible                       | Bulk / low-stakes lanes    |
| `blockrun/premium` | Best quality regardless of price        | Coding, architecture       |
| `blockrun/free`    | 100% free models, no balance            | Formatting, classification |

### Clean model names — no picker required to switch

A lot of the friction in [#80521](https://github.com/openclaw/openclaw/issues/80521) is that switching models means wrestling with long, mangled provider IDs. ClawRouter resolves short, memorable aliases so you can switch in one word:

```
sonnet   → Anthropic Claude Sonnet
opus     → Anthropic Claude Opus
flash    → Google Gemini Flash
grok     → xAI Grok
gpt5     → OpenAI GPT-5
```

```bash
/model grok        # switch this conversation to Grok
/model blockrun/eco  # or switch to a whole cost profile
```

No double-prefixes, no `provider/provider/model` confusion — the kind of model-ID mangling that fills the issue tracker (covered in detail in our [OpenRouter comparison](./clawrouter-vs-openrouter-llm-routing-comparison.md)).

### A poor-man's allowlist — `/exclude`

Full per-account admin allowlists ([#65557](https://github.com/openclaw/openclaw/issues/65557)) are an OpenClaw-side feature, but you can get most of the practical benefit now: `/exclude` removes models from routing entirely, including from every fallback chain.

```bash
/exclude openai/gpt-5         # this machine will never route to GPT-5
```

That's an effective cost-tier guardrail — the models you exclude are simply unreachable, no matter what an agent or skill requests.

### A cost-safe default — no surprise premium first turn

The default ClawRouter experience inverts [#88371](https://github.com/openclaw/openclaw/issues/88371). A fresh install defaults to `blockrun/auto` (cost-optimized) and the free tier works **with no balance at all** — so a brand-new user's first message routes to a cheap or free model by default, not silently onto a premium paid API. There's no "you just spent money and weren't told" first turn.

---

## Putting it together

A practical multi-lane setup:

1. **Default everything to `blockrun/auto`** — per-task routing for free, no config.
2. **Promote the lanes that need quality** to `blockrun/premium` per agent in `openclaw.json`.
3. **Demote the lanes that don't** to `blockrun/free`.
4. **`/exclude` the models you never want reached**, as a hard cost guardrail.
5. **Switch ad hoc** with `/model <alias>` when a single conversation needs something specific.

The goal isn't to pick a model — it's to stop picking the _wrong_ one for the task in front of you. Automatic routing handles the common case; per-agent profiles and aliases handle the rest.

---

## Related documentation

- [Routing Profiles](./routing-profiles.md) — `auto` / `eco` / `premium` / `free` reference
- [Smart LLM Router: 15-Dimension Classifier](./smart-llm-router-14-dimension-classifier.md) — how per-task classification works
- [We Read 100 OpenClaw Issues About OpenRouter](./clawrouter-vs-openrouter-llm-routing-comparison.md) — model-ID mangling and clean aliases
- [Configuration](./configuration.md) — per-agent config reference
- [Report a problem or request a feature](https://github.com/BlockRunAI/ClawRouter/discussions)
