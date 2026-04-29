# Single Trade Walkthrough

Reference for the `etoro-trading-assistant` skill. Load this when the user requests a **single trade** on a regular eToro account — *"buy $1,000 of AAPL"*, *"close my MSFT position"*, *"set a limit order on TSLA at $200"*. For multi-position bulk actions or anything agent-portfolio-related, see the `etoro-agent-portfolios` skill instead.

This walkthrough threads the other references in this skill (`api-conventions.md`, `account-snapshot.md`, `id-resolution.md`, `sso-and-session.md`) into one end-to-end flow. Where this walkthrough conflicts with another reference, the other reference wins — this is an integration map, not the authoritative source.

Source guides:

- <https://api-portal.etoro.com/guides/get-instrument-id>
- <https://api-portal.etoro.com/guides/market-orders>

---

## Account context

This walkthrough applies to **both regular eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the regular-account default.

> **Agent-portfolio override:** if you reached this walkthrough from the `etoro-agent-portfolios` skill, apply that skill's user-facing-numbers rule — replace dollar amounts with **percentages of equity** in every user-facing message (intent confirmation, error messages, outcome reports). Internally the API still takes USD `Amount` values; convert via `amount_usd = pct × equity`. The endpoint shapes, validation rules, and timing are unchanged.

---

## When to use this walkthrough

- A single market-open or market-close from the user.
- A single limit-order setup.
- The user's request fits in one trade. ("Buy AAPL, MSFT, and TSLA at 5% each" → that's a bulk action, load `etoro-agent-portfolios/references/bulk-trading.md` instead.)

---

## The flow

### Step 1 — Capture the user's intent

You need four things from the user. Ask explicitly if any are missing:

| Field | Example | Notes |
|---|---|---|
| **Action** | open / close / partial close / limit order | What the user wants done |
| **Symbol** | `AAPL` / `BTC` / `EURUSD` | The instrument |
| **Size** | `$1,000` (by amount) **or** `10` units (by units) | One or the other; not both |
| **Direction** | buy / sell | Required for opens; closes inherit from the position |

Optional fields — only include if the user explicitly asks:

- `Leverage` — default = 1 (per `api-conventions.md` "Leverage"). Send `1` explicitly even when not asked.
- `StopLossRate` / `TakeProfitRate` — for limit orders, see the SL/TP > 0 rule in `api-conventions.md`.

### Step 2 — Confirm environment (first trade in the session)

Before the first trade in the session, ask whether the user means **real** or **demo**. The endpoint paths differ (`/trading/execution/...` vs `/trading/execution/demo/...`) and the consequence of mismatched intent is obvious. Once confirmed, remember the choice for the rest of the session unless the user says otherwise.

### Step 3 — Resolve the instrument ID

If the symbol is already in your conversation cache (see `id-resolution.md` §1), use the cached `instrumentId`. Otherwise resolve live:

```
GET /market-data/search?internalSymbolFull=<SYMBOL>
```

- The response is `{ items: [{ instrumentId, internalSymbolFull, ... }, ...] }`.
- **Find the exact match** on `internalSymbolFull` — the search may return partial matches (`MSFT`, `MSFT.RTH`, `MSFT.EUR`). Confirm the field equals the requested symbol exactly before proceeding.
- If no exact match, ask the user to clarify — don't guess.
- **Cache the resolved ID in working memory** for the rest of the conversation.

Casing reminder: the search response uses `instrumentId` (lowercase `d`). See `api-conventions.md` for the per-endpoint casing table.

### Step 4 — Pre-flight (opens only)

For an **open**, verify the user has enough Available Cash before submitting:

```
GET /trading/info/{env}/pnl
```

Compute Available Cash via the formula in `account-snapshot.md` §1.

- If `requested_amount ≤ available_cash` → proceed.
- If `requested_amount > available_cash` → **stop and tell the user**, in absolute dollar amounts (e.g. *"You're trying to deploy $8,000 but only $5,000 is available."*). Don't auto-close other positions unless they explicitly ask. If they do ask to free cash, that's a rebalance trigger — load `etoro-agent-portfolios/references/rebalancing.md` "Insufficient-cash variant" for the structured close-then-cache-wait-then-open flow.

For a **close**, no cash pre-flight is needed — the position already exists.

### Step 5 — Confirm with the user

Show the planned trade in plain language and wait for confirmation:

```
Buying $1,000 of AAPL at market on your real account.
Leverage: 1× (no leverage).
Proceed?
```

Format the amount as the user originally specified — dollars or units — don't translate behind their back. For closes, include the position's **current value** (what they're about to liquidate, in dollars) so they have the right context for the decision:

```
Closing your MSFT position (currently worth $1,328).
Proceed?
```

Compute the current value as `position.amount + position.unrealizedPnL.pnL`. (Both fields are USD; this works for unleveraged and leveraged positions — see `account-snapshot.md` §3 for why.) Don't lead with units or with the P&L percentage on a close confirmation — those are useful in a status report, but the current dollar value is what matters when the user is deciding whether to liquidate.

Skip confirmation only when the user has already opted into "execute automatically for this session."

### Step 6 — Execute

#### Open by amount (cash)

```
POST /trading/execution/{env}/market-open-orders/by-amount
```

```json
{
  "InstrumentID": <resolved_id>,
  "IsBuy": true,
  "Leverage": 1,
  "Amount": 1000
}
```

#### Open by units

```
POST /trading/execution/{env}/market-open-orders/by-units
```

```json
{
  "InstrumentID": <resolved_id>,
  "IsBuy": true,
  "Leverage": 1,
  "Units": 10
}
```

Use **by-amount** for "buy $X" and "buy 5% of equity" intents (most common — translate the percentage to a dollar amount internally first via `amount_usd = pct × equity`). Use **by-units** when the user specifies an exact volume ("buy 10 shares of AAPL", "buy 1.5 BTC").

#### Close (full or partial)

```
POST /trading/execution/{env}/market-close-orders/positions/{positionId}
```

```json
{ "UnitsToDeduct": null }
```

- `UnitsToDeduct: null` → full close (entire position liquidated).
- `UnitsToDeduct: <number>` → partial close (only that many units closed; the rest stays open).
- The `positionId` lives in `clientPortfolio.positions[]` from the PnL endpoint — look it up by `instrumentID` first if the user named the position by symbol. If the user has multiple positions on the same instrument, ask which one (e.g. by `openDateTime`).

#### Limit order (market-if-touched)

```
POST /trading/execution/{env}/limit-orders
```

See `api-conventions.md` "Limit orders" for:

- The `Rate` must be **better than** the current price (lower for Buy, higher for Sell).
- `StopLossRate` and `TakeProfitRate` **must be > 0** even if the user doesn't want SL/TP — provide extreme defaults (`0.0001` for the losing side, `Rate * 100` for the winning side).

#### Send only what the user specified

Per `api-conventions.md` "Other optional parameters", omit anything the user didn't ask for. The single exception is `Leverage: 1` — send it explicitly even when unspecified, to avoid accidental leveraged positions if eToro ever changes the default for a particular instrument.

#### At-most-once: never retry on ambiguity

Trade-execution POSTs are not idempotent. If the response is **2xx with an order ID**, the trade is done — move on. If it's an **explicit error** (4xx other than 429, or 5xx with a body), log it and stop — *do not retry the same payload*. If the outcome is **ambiguous** (timeout, connection reset, no response, parse error), assume the trade may have landed: **do not retry** — go straight to Step 7 verification and check `positions[]` / `ordersForOpen[]`. Only **429** is a safe-to-retry response (the server explicitly told you it didn't process the request).

A duplicate position is far worse than a missing one — the missing case is recoverable by deliberately re-placing after verification; the duplicate has already executed. Full classification table and rationale: `bulk-trading.md` §4 "Response classification — at-most-once delivery".

#### Sizing — stated percentages are CEILINGS

When the user expresses intent as a percentage (*"buy 5% of equity in BTC"*) — common on agent-portfolios, occasional on regular accounts — the stated percentage is an **upper bound**, not a target. Apply the same anchor-and-floor rule from `bulk-trading.md` §§ 2 + 4:

```
EQUITY_ANCHOR = equity from a fresh /pnl read at the start of this trade
amount_usd    = floor(pct × EQUITY_ANCHOR × 100) / 100
```

Send `Amount: amount_usd` exactly. Never round up to a "nicer" number. Since eToro has no amount-slippage, the resulting `position.amount / EQUITY_ANCHOR` lands at or just under the stated cap — never above. After Step 7 verification, if `position.amount` somehow exceeds the floored expected amount, treat it as an agent-side bug and offer a corrective partial close (per `bulk-trading.md` §5 "Over-allocation check"). For dollar-form intents the same discipline applies in mirror image — *"buy $300 of BTC"* means send `$300` exactly, not `$305` for cleanliness.

### Step 7 — Verify the order landed

A successful POST means the order was **accepted**, not necessarily filled. Three landing states are possible:

| State | Where it lives in `clientPortfolio` | Tell the user |
|---|---|---|
| **Filled** | `positions[]` (look for matching `instrumentID`) | "Position opened at $X." (or for closes: "Position closed.") |
| **Pending market open** | `ordersForOpen[]` (matching `instrumentID`) | "Order accepted; will fill when the market opens." Common for stocks placed out of trading hours, weekends, etc. |
| **Failed / unknown** | Neither | "Order didn't land — please re-try or adjust." Surface any error message you got back from the POST. |

If the failure was a **401** (the credential is no longer valid — typically the user revoked their `x-user-key`), don't categorize it as a generic "failed trade" — surface it specifically per `sso-and-session.md` §3 and stop. The single-trade context makes hallucination less catastrophic than in a bulk flow, but the user still needs to know clearly that **no trade was placed** and that their key needs replacing.

#### Read-back timing

- For a **single open**: read `/pnl` immediately after the POST returns. `positions[]` typically updates promptly for fills; the 60-second cache mostly affects derived values like account-level P&L. If the position is in `positions[]`, you're done.
- For a **single close**: confirm the position is **gone** from `positions[]` (full close) or has **smaller `units`** (partial close). If neither happened after a re-read, treat as failed.
- **If you'll execute another trade right after that depends on freed cash**: wait the full 60 seconds before reading `/pnl` again — Available Cash relies on `credit` and `ordersForOpen` aggregation, which are subject to the cache. (This pattern usually means you're about to enter the rebalance flow — load `etoro-agent-portfolios/references/rebalancing.md`.)

### Step 8 — Report back

Tell the user the outcome in their original framing — **dollars and units, not percentages of equity**. (Percentages of equity are only used for agent-portfolios, per that skill's user-facing-numbers rule. Regular accounts always use absolute dollar amounts.)

- *"Bought $1,000 of AAPL at $179.50 (5.6 units)."*
- *"Order accepted — will fill when the US market opens (in ~3 hours)."*
- *"Closed your MSFT position. Realized P&L: +$28 (+2.1%)."*

Include side effects when meaningful (e.g. *"Available cash dropped from $8,000 to $3,000."*). Users appreciate the context — it shortens the next round of questions.

---

## Common gotchas

- **Don't conflate market-open and limit-order endpoints.** Market-open executes immediately at the current market price; limit-order sits dormant until the trigger price is hit (then converts to a market order).
- **Trade-execution request-body field is `InstrumentID`** (capital `D`) — same convention as the PnL response. The official guide example shows lowercase `d` (`InstrumentId`), but in practice the API expects capital `D`. The search response separately uses `instrumentId` (lowercase `d`), but that's a *response* field, not a request field.
- **Closing requires `positionId`, not `instrumentId`.** A position is a specific line-item in the portfolio. If the user has two AAPL positions (e.g. opened at different times), you need to pick the right one or close them both explicitly.
- **Don't synthesize the order's fill price from PnL.** Read it from the position itself once filled. Computing it from `unrealizedPnL.pnL / units` will drift from eToro's own valuation (see `account-snapshot.md` §3).
- **`/trading/info/trade/history` is real-only.** If the user asks "what did I trade this week" on a demo account, the endpoint returns `InsufficientPermissions` — surface a specific message rather than a generic auth error.

## Sanity checks

- [ ] Environment confirmed (real or demo) before the first trade in the session.
- [ ] Symbol resolved to a verified `instrumentId`; exact-match check on `internalSymbolFull` performed.
- [ ] For opens: Available Cash computed via `account-snapshot.md` §1 and verified ≥ requested amount.
- [ ] For closes: `positionId` looked up from `positions[]`, not assumed; if multiple positions exist on the same instrument, the user picked which one.
- [ ] User confirmed the trade in plain language before POST (unless auto-execute is opted in).
- [ ] `Leverage: 1` sent explicitly when not specified.
- [ ] **For percentage intents: `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100`** with `EQUITY_ANCHOR` from a fresh `/pnl` read at trade start; never rounded up; the stated percentage is a CEILING.
- [ ] **At-most-once: only 429 is retried; 4xx/5xx and ambiguous outcomes are reconciled by reading `positions[]`, never by re-firing the same payload.**
- [ ] Post-execution: `/pnl` re-read and trade categorized as filled / pending / failed; pending-market-open distinction surfaced. For percentage intents, `position.amount ≤ floor(pct × EQUITY_ANCHOR × 100) / 100`; over-fill triggers a corrective partial close (per `bulk-trading.md` §5).
- [ ] Outcome reported in the user's original framing (their dollars or their units), with meaningful side effects (e.g. cash impact) noted.

## References used by this walkthrough

- `api-conventions.md` — auth, casing, demo/real paths, limit-order SL/TP rules, leverage default, optional-parameter handling.
- `id-resolution.md` — symbol → `instrumentId` resolution and conversation-scoped caching.
- `account-snapshot.md` — Available Cash formula and the rest of the PnL response shape.
- `sso-and-session.md` — handling 401s during execution.

For multi-position actions (build a portfolio, rebalance, conditional/triggered rules), don't use this walkthrough — load the `etoro-agent-portfolios` skill.
