# eToro Skills for Runtime Agents

> Drop-in markdown skills that teach an LLM agent to trade safely on the eToro Public API — single trades, bulk portfolio builds, rebalancing, conditional rules, and the agent-portfolios product, with the behavioral safety norms baked in.

A skill package for **LLM agents that act on behalf of an end user** when interacting with the eToro Public API — for example, a chatbot inside an app where the user says *"rebalance my portfolio"* and the agent executes the trades directly.

The skills are plain markdown with `SKILL.md` entry points and supporting `references/` pages — deployable into any agent runtime that loads markdown skills (Claude Skills, OpenAI custom GPTs, in-app agents, etc.). Output is conversational: trade confirmations, execution results, and status reports — not code.

## Skills included

- **`etoro-trading-assistant`** — the foundational runtime skill for any agent talking to the eToro Public API. Covers identity, behavioral norms, and **all execution workflows** (single trade, bulk portfolio build, rebalancing, conditional/triggered rules) plus the foundational eToro API knowledge (request conventions, account-snapshot formulas, ID resolution, session handling). Includes `references/execution-invariants.md` — the four cross-cutting rules (anchor freeze, ceilings on allocations, at-most-once delivery, never-hallucinate-on-401) that every workflow applies — and `references/examples.md` with end-to-end conversation walkthroughs.

- **`etoro-agent-portfolios`** — narrow skill for the eToro **agent-portfolios product** (dedicated copy-traded accounts that mirror the user's main account proportionally). Covers only what's *different* about agent-portfolios: the product concept, the conversational onboarding flow (in `references/onboarding.md`), and two execution overrides — **Override A** (user-facing numbers are percentages of equity, never dollars) and **Override B** (always read live equity and cash from `/pnl` before every workflow). **Always loaded alongside `etoro-trading-assistant`** — agent-portfolio trade execution uses the workflow references in that skill, with these overrides applied throughout.

## How to use

1. **Always load `etoro-trading-assistant` as the foundational skill** for any runtime agent that touches the eToro API. It contains all the workflows (single trade, bulk, rebalance, conditional rules) plus the foundational API knowledge.
2. **Additionally load `etoro-agent-portfolios`** if the user's request involves an agent-portfolio (creating one, operating one, copying-trading via one). The agent-portfolios skill is small — it carries only what changes when running against an agent-portfolio (display format override, onboarding flow). The actual execution still uses `etoro-trading-assistant`'s workflow references.
3. Inside each skill, the `SKILL.md` is the entry point; the `references/` directory contains deeper material that gets loaded when the agent's task matches.

### Mental model

The two account types are **main accounts** (the user's primary eToro account, can be either real or demo per the API key's environment scope) and **agent-portfolios** (dedicated copy-traded accounts, always real-environment). Both are real eToro accounts holding real money in production — the contrast is *which account*, not real-vs-fake.

The execution workflows (single trade, bulk build, rebalance, conditional rules) work identically on both. The only meaningful differences:

- **Display format**: main accounts use absolute dollar amounts (and units where relevant); agent-portfolios use percentages of equity. The agent-portfolios skill enforces this override.
- **Onboarding**: agent-portfolios have a creation flow; main accounts use the user's existing API key directly.
- **API user key (`x-user-key`)**: main accounts use the user's own main-account API user-key; agent-portfolios use the agent-portfolio's `userToken` from the creation flow.
- **Environment**: a main-account user-key is bound to one environment (real or demo) at creation — the agent determines which by probing, never asks the user. Agent-portfolios are always real.

Everything else — endpoints, headers (including the canonical `x-api-key`), rate limits, the 10-second PnL cache, validation rules, retry strategies — is identical.

## Official eToro API documentation

The skills here curate the workflows and conventions a runtime agent needs day-to-day. For everything else, the source of truth is the official eToro Developer Portal:

- **API portal:** <https://api-portal.etoro.com/>
- **LLM-friendly index:** <https://api-portal.etoro.com/llms.txt>
- **MCP server (for runtimes that support MCPs):** <https://api-portal.etoro.com/mcp>

Consult the portal when a response shape doesn't match the references here, when the user asks about a capability not covered by these skills (e.g. deeper watchlist, feeds, or comments operations), or when you need the full OpenAPI spec for an endpoint. When the portal and a reference here disagree, prefer the portal.

## Layout

```
external-agent-skills/
├── README.md
└── skills/
    ├── etoro-trading-assistant/
    │   ├── SKILL.md
    │   └── references/
    │       ├── execution-invariants.md      (anchor freeze, ceilings, at-most-once, never-hallucinate-on-401)
    │       ├── api-conventions.md           (hosts, headers, casing, rate limits, response shape, trade defaults)
    │       ├── account-snapshot.md          (PnL endpoint formulas + per-field interpretation)
    │       ├── id-resolution.md             (symbol → instrumentID, conversation-scoped caching)
    │       ├── sso-and-session.md           (identity from /me, dead-session detection, "Reconnect" flow)
    │       ├── single-trade-walkthrough.md  (end-to-end single trade)
    │       ├── bulk-trading.md              (multi-position build)
    │       ├── rebalancing.md               (current → target with phase orchestration)
    │       ├── conditional-rules.md         (triggered/rule-based trading)
    │       └── examples.md                  (worked end-to-end conversation walkthroughs)
    └── etoro-agent-portfolios/
        ├── SKILL.md                         (concept + Override A + Override B + cross-refs)
        └── references/
            └── onboarding.md                (key collection, portfolio creation, secret-token handoff)
```

The `etoro-agent-portfolios` skill carries only the concept, the two overrides, and the onboarding flow — every trade-execution workflow is loaded from `etoro-trading-assistant/references/`, with the overrides applied to user-facing output and live `/pnl` reads.
