---
name: etoro-agent-portfolios
description: Runtime skill for the eToro agent-portfolios product — dedicated copy-traded accounts where positions opened on the agent-portfolio are mirrored proportionally into the user's real account based on the user's `investmentAmountInUsd`. Covers the conversational onboarding flow (key collection with type detection, portfolio creation, key handoff) plus the user-facing-numbers override (percentages of equity, never dollars) and the always-check-equity-first execution rule. MUST BE LOADED ALONGSIDE `etoro-trading-assistant` — agent-portfolio trade execution uses that skill's workflow references (single trade, bulk trading, rebalancing, conditional rules) with this skill's overrides applied. Use when an end user wants to create an agent-portfolio, list existing ones, or have the agent execute trades on an agent-portfolio's behalf.
---

# eToro Agent Portfolios

Use this skill when the user wants to **create or operate an agent-portfolio**. This skill assumes `etoro-trading-assistant` is also loaded — agent-portfolio trade execution uses the same workflows (single trade, bulk build, rebalancing, conditional rules) as a regular account, just with this skill's display-format override.

> ⚠️ **Don't try to handle agent-portfolio trades using only this skill.** All execution flows live in `etoro-trading-assistant`. This skill carries only what's *different* about agent-portfolios — which is just: the product concept, the onboarding flow, and the user-facing-numbers rule.

## What agent-portfolios share with regular accounts

Trade execution against an agent-portfolio uses identical infrastructure to a regular account: same Public API endpoints, same headers (`x-request-id`, hardcoded `x-api-key`), same rate limits (20 req/min on trade-execution endpoints), same 60-second PnL cache, same single-trade and bulk and rebalance and conditional-rule workflows. **Load `etoro-trading-assistant` and follow its references for execution.**

The only execution difference: the `x-user-key` header carries the agent-portfolio's `userToken` (from creation, see Step 3 below) instead of the user's main account API user-key. Everything else is unchanged.

## What's different about agent-portfolios

Three things, plus the onboarding flow:

1. The **product concept** (mirrored copy-trading; the agent-portfolio is the *source* portfolio whose trades are mirrored proportionally into the user's real account).
2. The **user-facing-numbers rule** (percentages of equity, never dollars).
3. The **always-check-equity-first execution rule** — before any workflow, read live equity and cash from `/pnl`; never assume a fixed value.
4. The **onboarding flow** (Steps 1–3 below: collect key → detect type → create portfolio).

Once an agent-portfolio is created and you have its `userToken`, you operate on it through `etoro-trading-assistant` exactly as you would a regular account — *except* you apply the user-facing-numbers and always-check-equity rules to every workflow.

## The agent-portfolio concept

A dedicated eToro account that the agent operates on its own credentials. The agent-portfolio has its own equity, sized by eToro at creation time — **read the actual value live from `/pnl`; do not assume a fixed amount.** That equity figure (and any per-position dollar amounts derived from it) is operational context for the agent's sizing decisions, not user-facing information.

When the user creates one, they specify an `investmentAmountInUsd` which is **real money deducted from their real eToro account** to copy-trade the agent-portfolio. Positions opened by the agent-portfolio are mirrored proportionally into the user's real account: if the agent-portfolio allocates *N%* of its equity to AAPL, the user's mirror allocates *N%* of their `investmentAmountInUsd` to AAPL. The agent-portfolio's absolute equity and the user's investment amount are independent inputs — the proportion is what mirrors.

The agent trades on behalf of the agent-portfolio using **the agent-portfolio's own user token**. All endpoints are the **real** Public API endpoints (no `/demo/` segment); agent-portfolios are real accounts.

## CRITICAL — always check current equity/cash first

This is the strict, agent-portfolio-specific application of the general "Check current equity / cash before any trade workflow" rule in `etoro-trading-assistant`'s SKILL.md. It's **non-negotiable** because user intents are expressed as percentages and the only way to size them correctly is to read the live equity.

Before starting **any** workflow on an agent-portfolio (single trade, bulk build, rebalance, conditional rule, status report), call:

```
GET /trading/info/real/pnl
```

with the agent-portfolio's `userToken` as `x-user-key`. Use the formulas in `etoro-trading-assistant`'s `references/account-snapshot.md` to compute Available Cash and Equity. These numbers are essential **internal context** — you need the equity in dollars to compute the `Amount` field for any API call (a "5% in AAPL" intent becomes `Amount = 0.05 × equity_usd`) — but they are **never** disclosed to the user.

Don't reuse equity values from earlier in the conversation; positions move and the equity changes. Re-read `/pnl` at the start of each workflow.

## CRITICAL — user-facing numbers rule (the override)

**Never show the user absolute dollar amounts** from the agent-portfolio's equity or position sizes. Always translate to **percentages of equity** in user-facing output:

| Show this | NOT this |
|---|---|
| "+2.1%" | "+$34.70" |
| "75% cash" | "$7,500 cash" |
| "Invest 2.5% in AAPL" | "Invest $250 in AAPL" |

This rule **overrides the regular-account dollar default** in every workflow loaded from `etoro-trading-assistant`. The override applies to:

- `single-trade-walkthrough.md` — replace dollars with percentages in user-facing intent confirmations, error messages, and outcome reports.
- `bulk-trading.md` — use percentages in plan confirmations and post-execution reports.
- `rebalancing.md` — use percentages in diff confirmations and the trade-off communication for the insufficient-cash variant.
- `conditional-rules.md` — express rule sizes, targets, and trigger notifications in percentages.

Internal API calls still use dollar `Amount` fields — convert at the call site as `amount_usd = pct × equity`. The user never sees the dollar number.

Each workflow reference has an "Account context" callout near the top reminding the agent of this override. When you load one of those references after this skill, **apply the override consistently**.

## Step 1 — Collect a user key

Ask the user for an **API user key** (`x-user-key`). Offer both options together, agent-portfolio key listed first (preferred):

1. **Agent-portfolio API key** (preferred) — created specifically for an existing agent-portfolio.
2. **Main account API key** — created with **Environment: Real** and **Write Access** permission at <https://www.etoro.com/settings/trade>.

The `x-api-key` header is the canonical eToro partner key from `etoro-trading-assistant`'s `references/api-conventions.md` — do not ask the user for it.

### Detect the key type

After receiving the key, call:

```
GET https://public-api.etoro.com/api/v1/agent-portfolios
```

| Response | Key type | Next step |
|---|---|---|
| **200** | Main-account key with `real:write` | Step 2 — create a new portfolio |
| **403 `Forbidden`** | Agent-portfolio key | Check portfolio state (below), then jump to operations (load `etoro-trading-assistant` workflows) |
| **403 `InsufficientPermissions`** | Key without `real:write` | Ask the user for a key with the right scope |

Skip Steps 2 and 3 silently when the key is already an agent-portfolio key — don't tell the user you're skipping steps.

### Check existing-portfolio state (agent-portfolio key only)

Call `GET /trading/info/real/pnl`. (For the response shape and aggregation formulas, see `etoro-trading-assistant`'s `references/account-snapshot.md`.)

- **Empty** (no `positions[]` and no `ordersForOpen[]`): tell the user the portfolio is empty and ask if they'd like to build one. (For a build, load `etoro-trading-assistant`'s `references/bulk-trading.md` and apply the percentages override.)
- **Active**: report the number of open positions and pending orders (in percentages of equity), then ask if they want to modify it.

## Step 2 — Gather portfolio parameters (new portfolio only)

| Parameter | Required | Notes |
|---|---|---|
| **Portfolio name** | Yes | 6–10 characters, unique. Maps to `agentPortfolioName`. |
| **Investment amount** | Yes | USD; real funds drawn from the user's real account. Maps to `investmentAmountInUsd`. (This is one of the few places dollars appear in user-facing language — it's the user's *own real account* funds being committed, not the agent-portfolio's internal balance.) |

Auto-generate (don't ask the user):

- **`userTokenName`**: lowercase `agentPortfolioName` + `-key-` + 6 random digits, e.g. `portfoliox-key-482917`.
- **`scopeIds`**: always `[202]` (real:write).

## Step 3 — Create the agent-portfolio

```
POST https://public-api.etoro.com/api/v1/agent-portfolios
```

```json
{
  "investmentAmountInUsd": <amount>,
  "agentPortfolioName": "<name>",
  "userTokenName": "<auto-generated>",
  "scopeIds": [202]
}
```

On `201`, the response carries:

- `agentPortfolioId`, `agentPortfolioName`, `agentPortfolioGcid`
- `agentPortfolioVirtualBalance` — the agent-portfolio's starting equity (whatever value eToro sets; **don't show this to the user**, and don't depend on a specific value being returned)
- `mirrorId`
- `userTokens[0].userToken` — **the secret token, only available at creation time**
- `userTokens[0].userTokenId`, `userTokens[0].clientId`

**Critical:** present the `userToken` to the user immediately, labelled with the auto-generated `userTokenName`:

> *"Here is your key **portfoliox-key-482917**. Store it securely — this is the only time it will be shown. If lost, you'll need to create a new key for this agent-portfolio."*

## Operating an agent-portfolio (after creation)

Once the agent-portfolio exists and you have its `userToken`, all trading flows live in `etoro-trading-assistant`:

| User wants to | Load |
|---|---|
| Make a single trade (open / close / partial close / limit order) | `etoro-trading-assistant`'s `references/single-trade-walkthrough.md` |
| Build a multi-position portfolio | `etoro-trading-assistant`'s `references/bulk-trading.md` |
| Rebalance to a target | `etoro-trading-assistant`'s `references/rebalancing.md` |
| Set up triggered/conditional rules | `etoro-trading-assistant`'s `references/conditional-rules.md` |
| See current account state | `etoro-trading-assistant`'s `references/account-snapshot.md` |

For all of the above:

- Use the agent-portfolio's `userToken` as `x-user-key`.
- **Always read current equity/cash from `/pnl` first** before any sizing or execution (see the always-check-equity-first rule above). Don't assume the agent-portfolio's equity equals what it started with at creation.
- Apply the **user-facing-numbers rule** (percentages of equity, not dollars) per the override note in each reference.
- Approval-mode handling is the same as for regular accounts — default to "ask before each trade" unless the user opts into auto-execute (e.g. for recurring rebalancing).

## Retrieving existing agent-portfolios

```
GET https://public-api.etoro.com/api/v1/agent-portfolios
```

Returns `agentPortfolios[]`, each with `agentPortfolioId`, `agentPortfolioName`, `agentPortfolioGcid`, `agentPortfolioVirtualBalance`, `mirrorId`, `createdAt`, and `userTokens[]` (without the secret `userToken` value — that's only returned at creation time).

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
| Agent-portfolio equity | The agent-portfolio's own funds, varies over time as positions move. Read live from `/pnl` at the start of every workflow — never assume a fixed value. **Hidden from the user**; used only as internal context for sizing. |
| `investmentAmountInUsd` | Real money deducted from the *user's real account* to copy-trade this portfolio. The one place where a dollar amount appears in user-facing language (it's the user's own real funds being committed at creation). |
| Proportional mirroring | The agent-portfolio's allocation percentage equals the user's mirror allocation percentage. If the agent-portfolio allocates 5% of its equity to AAPL, the user's mirror allocates 5% of their `investmentAmountInUsd` to AAPL. The agent-portfolio's absolute equity and the user's investment are independent inputs. |
| User token | Secret created once at portfolio creation. User must store it themselves. |
| `scopeIds` | `202` = real:write (default and required for trading). |

## Error handling (creation-specific)

| Code | Meaning | Action |
|---|---|---|
| 400 | Validation failed (name length, investment minimum) | Show the error; ask the user to correct input. |
| 401 | Unauthorized | Verify both keys and required scopes. See `etoro-trading-assistant`'s `references/sso-and-session.md` for full handling. |
| 207 | Portfolio created but user-token issuance failed | Portfolio exists but needs a new token — inform the user. |
| 500 | Server error | Retry once, then surface to the user. |

For trade-execution errors (429 rate limit, 401 mid-session, etc.), see the relevant workflow reference in `etoro-trading-assistant`.

## When the `userToken` stops working (401)

If a Public API call using an agent-portfolio's `userToken` returns 401 Unauthorized, the `userToken` has been revoked or invalidated — same behavior as a regular-account user-key being revoked. There is **no refresh** for an agent-portfolio `userToken`.

Recovery:

1. **Stop the current workflow immediately.** Don't keep firing requests against the dead token; never report a workflow as completed when it failed mid-way.
2. Tell the user the `userToken` is no longer valid and ask them to:
   - Confirm whether they intentionally revoked the agent-portfolio's key.
   - **If the agent-portfolio still exists** but the `userToken` is lost: they may need to create a new agent-portfolio (the original `userToken` was the only one, returned at creation time only — it cannot be re-issued for the same agent-portfolio).
   - **If the user wants to keep operating** the same allocation strategy, they would need to re-create a new agent-portfolio and re-run the build flow.

See `etoro-trading-assistant`'s `references/sso-and-session.md` §§3–4 for the full 401 handling pattern, including the don't-hallucinate-trades rule.

## Sanity checks

- [ ] Every user-facing number is a percentage of equity, never a dollar amount from the agent-portfolio's equity or position sizes.
- [ ] Before each workflow, current equity and Available Cash are read live from `/pnl` — not assumed, not carried over from earlier in the conversation.
- [ ] The hardcoded `x-api-key` (per `api-conventions.md`) is used; the user is never asked for it.
- [ ] `userToken` is presented to the user exactly once at portfolio creation, with explicit storage instructions.
- [ ] After creation, all execution flows are loaded from `etoro-trading-assistant` (single trade, bulk, rebalance, conditional rules) — not handled inline here.
- [ ] The percentages-of-equity override is applied to every output from those flows.
- [ ] No specific dollar amount (e.g. "$10,000") is mentioned to the user as the agent-portfolio's balance or starting capital — the value comes from `/pnl` and is operational only.
- [ ] On 401 against the agent-portfolio's `userToken`, all workflows stop immediately; the user is informed clearly that the token is revoked; trades that didn't execute are not reported as if they did.
