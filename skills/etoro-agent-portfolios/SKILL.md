---
name: etoro-agent-portfolios
description: Create and operate eToro agent-portfolios — dedicated copy-traded accounts whose trades are mirrored proportionally into the user's main eToro account. Covers the onboarding flow (API key collection, portfolio creation, secret-token handoff) and the agent-portfolio-specific execution overrides (percentages of equity in user-facing output instead of dollars; freeze live equity and cash before every workflow). Use when the user mentions agent-portfolios, asks to create or set up an agent-portfolio, list existing agent-portfolios, copy-trade through one, or have the agent operate trades on an agent-portfolio's behalf. Load alongside `etoro-trading-assistant` — that skill carries the actual trade-execution workflows.
---

# eToro Agent Portfolios

Use this skill when the user wants to **create or operate an agent-portfolio**. This skill assumes `etoro-trading-assistant` is also loaded — agent-portfolio trade execution uses the same workflows (single trade, bulk build, rebalancing, conditional rules) as a main account, with two agent-portfolio-specific overrides applied throughout.

> ⚠️ **Don't try to handle agent-portfolio trades using only this skill.** All execution flows live in `etoro-trading-assistant`. This skill carries only what's *different* about agent-portfolios — the product concept, the onboarding flow (in `references/onboarding.md`), and the two execution overrides below.

## What's different about agent-portfolios

Trade execution against an agent-portfolio uses identical infrastructure to a main account: same Public API endpoints, same headers, same rate limits, same 60-second PnL cache, same workflows (single trade, bulk, rebalance, conditional rules). The only execution difference is that the `x-user-key` header carries the agent-portfolio's `userToken` (from creation; see `references/onboarding.md`) instead of the user's main-account key.

What's specific to agent-portfolios:

1. The **product concept** (mirrored copy-trading; see below).
2. The **onboarding flow** — `references/onboarding.md` (key collection → type detection → portfolio creation → secret-token handoff).
3. **Override A — percentages of equity in user-facing output** (never dollars). Applies to every user-facing message in every workflow loaded from `etoro-trading-assistant`.
4. **Override B — always read live equity and cash from `/pnl` before every workflow.** This is a hard rule on top of the general anchor-freeze invariant in `etoro-trading-assistant/references/execution-invariants.md` §1; see "Override B" below for why it's specifically critical here.

## The agent-portfolio concept

A dedicated eToro account that the agent operates on its own credentials. The agent-portfolio has its own equity, sized by eToro at creation time — **read the actual value live from `/pnl`; do not assume a fixed amount.** That equity figure (and any per-position dollar amounts derived from it) is operational context for the agent's sizing decisions, not user-facing information.

When the user creates one, they specify an `investmentAmountInUsd` which is **funds deducted from their main eToro account** to copy-trade the agent-portfolio. Positions opened by the agent-portfolio are mirrored proportionally into the user's main account: if the agent-portfolio allocates *N%* of its equity to AAPL, the user's mirror allocates *N%* of their `investmentAmountInUsd` to AAPL. The agent-portfolio's absolute equity and the user's investment amount are independent inputs — the proportion is what mirrors.

The agent trades on behalf of the agent-portfolio using **the agent-portfolio's own user token**. Agent-portfolios always live in the real environment — endpoints never carry the `/demo/` segment.

## Override A — user-facing numbers are percentages of equity, never dollars

**Never show the user absolute dollar amounts** from the agent-portfolio's equity or position sizes. Always translate to **percentages of equity** in user-facing output:

| Show this | NOT this |
|---|---|
| "+2.1%" | "+$34.70" |
| "75% cash" | "$7,500 cash" |
| "Invest 2.5% in AAPL" | "Invest $250 in AAPL" |

This rule **overrides the main-account dollar default** in every workflow loaded from `etoro-trading-assistant`. Applies to:

- `single-trade-walkthrough.md` — replace dollars with percentages in user-facing intent confirmations, error messages, and outcome reports.
- `bulk-trading.md` — use percentages in plan confirmations and post-execution reports.
- `rebalancing.md` — use percentages in diff confirmations and the trade-off communication for the insufficient-cash variant.
- `conditional-rules.md` — express rule sizes, targets, and trigger notifications in percentages.

Internal API calls still use dollar `Amount` fields — convert at the call site as `amount_usd = pct × EQUITY_ANCHOR` (per the execution invariants — see Override B). The user never sees the dollar number.

The single exception is `investmentAmountInUsd` at portfolio creation — that's the user's *own main-account* funds being committed, so it's named in dollars (see `references/onboarding.md` Step 2).

## Override B — always read live equity and cash before every workflow

The general anchor-freeze rule in `etoro-trading-assistant/references/execution-invariants.md` §1 already says to read `/pnl` at workflow start and freeze `EQUITY_ANCHOR` + `CASH_ANCHOR`. **For agent-portfolios this is non-negotiable and specifically critical**, because user intents are expressed as percentages and the only way to size them correctly is to anchor on a live equity value — not a value remembered from earlier in the conversation, not the `agentPortfolioVirtualBalance` returned at creation, not anything else.

Before starting **any** workflow on an agent-portfolio (single trade, bulk build, rebalance, conditional rule, status report):

```
pnl = GET /trading/info/real/pnl   // x-user-key = the agent-portfolio's userToken
EQUITY_ANCHOR = equity(pnl)        // X% of intent → X% × EQUITY_ANCHOR
CASH_ANCHOR   = available_cash(pnl)
```

Use these for ALL sizing decisions in the workflow — `Amount = floor(pct × EQUITY_ANCHOR × 100) / 100` (per `execution-invariants.md` §2 ceilings). These numbers are **internal context only**; never disclosed to the user (see Override A).

A new workflow gets fresh anchors. Don't reuse anchors from earlier in the conversation.

## Operating an agent-portfolio (after onboarding)

Once the agent-portfolio exists and you have its `userToken` (per `references/onboarding.md`), all trading flows live in `etoro-trading-assistant`:

| User wants to | Load |
|---|---|
| Make a single trade (open / close / partial close / limit order) | `etoro-trading-assistant`'s `references/single-trade-walkthrough.md` |
| Build a multi-position portfolio | `etoro-trading-assistant`'s `references/bulk-trading.md` |
| Rebalance to a target | `etoro-trading-assistant`'s `references/rebalancing.md` |
| Set up triggered/conditional rules | `etoro-trading-assistant`'s `references/conditional-rules.md` |
| See current account state | `etoro-trading-assistant`'s `references/account-snapshot.md` |

For all of the above:

- Use the agent-portfolio's `userToken` as `x-user-key`.
- Apply Override A (percentages, never dollars) to user-facing output.
- Apply Override B (always read live equity/cash from `/pnl` first) — this is the agent-portfolio-specific application of the anchor-freeze invariant.
- Apply all other invariants from `execution-invariants.md` (ceilings on allocations, at-most-once delivery, never-hallucinate-on-401) unchanged.
- Approval-mode handling is the same as for main accounts — default to "ask before each trade" unless the user opts into auto-execute (e.g. for recurring rebalancing).

## Presenting the agent-portfolio to the user

When the user asks to see their agent-portfolio, show each position as a **weight (%) of total equity** and the instrument's display name. **Do not** show per-position dollar P&L; use percentage P&L if showing P&L at all.

```
Your agent-portfolio:
- BTC:   30%
- AAPL:  25%
- MSFT:  20%
- ETH:   15%
- Cash:  10%
```

## Key concepts

| Concept | Detail |
|---|---|
| Agent-portfolio equity | The agent-portfolio's own funds, varies over time as positions move. Read live from `/pnl` at the start of every workflow per Override B. **Hidden from the user** per Override A; used only as internal context for sizing. |
| `investmentAmountInUsd` | Funds deducted from the *user's main account* to copy-trade this portfolio. The one place where a dollar amount appears in user-facing language — it's the user's own main-account funds being committed at creation. |
| Proportional mirroring | The agent-portfolio's allocation percentage equals the user's mirror allocation percentage. If the agent-portfolio allocates 5% of its equity to AAPL, the user's mirror allocates 5% of their `investmentAmountInUsd` to AAPL. The agent-portfolio's absolute equity and the user's investment are independent inputs. |
| User token | Secret created once at portfolio creation. User must store it themselves (see `references/onboarding.md` Step 3). |
| `scopeIds` | `202` = real:write (default and required for trading). |

## Reference pages in this skill

- `references/onboarding.md` — the conversational onboarding flow (Steps 1–3: collect key, detect type, create portfolio + secret-token handoff), plus retrieval of existing agent-portfolios, creation-error handling, and `userToken`-revoked recovery.

For everything else (workflows, invariants, foundational API knowledge), see `etoro-trading-assistant`.

## Sanity checks

- [ ] **Override A — percentages, never dollars** — every user-facing number is a percentage of equity, not a dollar amount from the agent-portfolio's equity or position sizes.
- [ ] **Override B — live `/pnl` read before every workflow** — `EQUITY_ANCHOR` + `CASH_ANCHOR` frozen from a fresh `/pnl` (not from the conversation, not from `agentPortfolioVirtualBalance`, not from a previous workflow).
- [ ] All other invariants from `execution-invariants.md` apply unchanged (ceilings on opens, mirror-image rule for closes when freeing cash, at-most-once on every POST, stop on 401).
- [ ] The hardcoded `x-api-key` (per `api-conventions.md`) is used; the user is never asked for it.
- [ ] `userToken` is presented to the user exactly once at portfolio creation, with explicit storage instructions (`references/onboarding.md` Step 3).
- [ ] After creation, all execution flows are loaded from `etoro-trading-assistant` — not handled inline here.
- [ ] No specific dollar amount (e.g. "$10,000") is mentioned to the user as the agent-portfolio's balance or starting capital — the value comes from `/pnl` and is operational only.
