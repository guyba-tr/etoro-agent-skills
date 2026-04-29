# Single Trade Walkthrough

Reference for the `etoro-trading-assistant` skill. Load this when the user requests a **single trade** — *"buy $1,000 of AAPL"*, *"close my MSFT position"*, *"set a limit order on TSLA at $200"*. Applies to both main accounts and agent-portfolios. For multi-position bulk actions, see `bulk-trading.md`. For anything agent-portfolio-specific (onboarding, the percentages override), see the `etoro-agent-portfolios` skill.

This walkthrough threads the other references in this skill (`api-conventions.md`, `account-snapshot.md`, `id-resolution.md`, `sso-and-session.md`) into one end-to-end flow. Where this walkthrough conflicts with another reference, the other reference wins — this is an integration map, not the authoritative source.

Source guides:

- <https://api-portal.etoro.com/guides/get-instrument-id>
- <https://api-portal.etoro.com/guides/market-orders>

---

## Account context

This walkthrough applies to **both main eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the main-account default.

> **Agent-portfolio override:** if you reached this walkthrough from the `etoro-agent-portfolios` skill, apply **Override A** from that skill — replace dollar amounts with **percentages of equity** in every user-facing message (intent confirmation, error messages, outcome reports). Internally the API still takes USD `Amount` values; convert via `amount_usd = pct × EQUITY_ANCHOR`. The endpoint shapes, validation rules, and timing are unchanged.

---

## When to use this walkthrough

- A single market-open or market-close from the user.
- A single limit-order setup.
- The user's request fits in one trade. ("Buy AAPL, MSFT, and TSLA at 5% each" → that's a bulk action, load `bulk-trading.md` instead.)

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

### Step 2 — Resolve the instrument ID

If the symbol is already in your conversation cache (see `id-resolution.md` §1), use the cached `instrumentId`. Otherwise resolve live:

```
GET /market-data/search?internalSymbolFull=<SYMBOL>
```

- The response is `{ items: [{ instrumentId, internalSymbolFull, ... }, ...] }`.
- **Find the exact match** on `internalSymbolFull` — the search may return partial matches (`MSFT`, `MSFT.RTH`, `MSFT.EUR`). Confirm the field equals the requested symbol exactly before proceeding.
- If no exact match, ask the user to clarify — don't guess.
- **Cache the resolved ID in working memory** for the rest of the conversation.

Casing reminder: the search response uses `instrumentId` (lowercase `d`). See `api-conventions.md` for the per-endpoint casing table.

### Step 3 — Pre-flight (opens only)

For an **open**, verify the user has enough Available Cash before submitting:

```
GET /trading/info/{env}/pnl
```

Compute Available Cash via the formula in `account-snapshot.md` §1.

- If `requested_amount ≤ available_cash` → proceed (with the open-buffer check below).
- If `requested_amount > available_cash` → **stop, present the gap, and ask the user explicitly** what to do. Don't auto-close other positions to fund the trade — closes are destructive actions and require explicit consent (per `etoro-trading-assistant/SKILL.md` "Confirm before destructive actions"). Offer two options:

  ```
  You're trying to deploy $8,000 of AAPL but only $5,000 is available.
  Two options:

    1. Reduce the trade size to fit $5,000.
    2. Free up the missing $3,000 by closing or reducing some of your existing
       positions. I'll propose which positions to reduce and confirm with you
       before any closes happen.

  Which would you like?
  ```

  **Only if the user picks option 2** (or had already given a close-to-fund instruction up front, e.g. *"buy $8,000 of AAPL, close MSFT to cover it"*) load `rebalancing.md` "Insufficient-cash variant" for the structured close-then-cache-wait-then-open flow. In agent-portfolio context, frame both options in percentages.

For a **close**, no cash pre-flight is needed — the position already exists.

#### Open buffer — shrink the trade by 1% if cash would drop below 1% of equity

Per `execution-invariants.md` §2 "Open buffer", before sending the open:

```
post_trade_cash_pct = (CASH_ANCHOR − requested_amount) / EQUITY_ANCHOR

if post_trade_cash_pct < 0.01:
  requested_amount = floor(requested_amount × 0.99 × 100) / 100
  # User-facing disclosure required at Step 4 (see below)
```

**Worked example.** *"Buy $300 of BTC"* with `EQUITY_ANCHOR = $10,000` and `CASH_ANCHOR = $300` (almost no cash to spare). `post_trade_cash_pct = 0%` < 1% → buffer applies. Send `Amount: $297.00` and disclose the under-fill in the Step 4 confirmation. (For `EQUITY_ANCHOR = $10,000`, `CASH_ANCHOR = $5,000`, requesting `$300` of BTC: `post_trade_cash_pct = 47%` > 1% → no buffer; send `$300` exactly.)

### Step 4 — Confirm with the user

Show the planned trade in plain language and set the user's expectation that verification takes ~1 minute (per Step 6's read-back-timing rule), then wait for confirmation:

```
Buying $1,000 of AAPL at market.
Leverage: 1× (no leverage).

I'll place the order and confirm the result in about a minute.
Proceed?
```

**If the open buffer (Step 3) was applied to a dollar-form intent**, surface it explicitly in the confirmation so the user isn't surprised by a smaller-than-stated amount:

```
Buying $297 of BTC at market (1% under your stated $300 — keeping a small
cash cushion so eToro's per-trade fees don't push your cash below zero).
Leverage: 1× (no leverage).

I'll place the order and confirm the result in about a minute.
Proceed?
```

For percentage-form intents on a single trade (rare, but possible — e.g. *"buy 5% of equity in BTC"*), the under-fill is invisible at percentage resolution; no disclosure needed.

Format the amount as the user originally specified — dollars or units — don't translate behind their back. For closes, include the position's **current value** (what they're about to liquidate, in dollars) so they have the right context for the decision:

```
Closing your MSFT position (currently worth $1,328).
I'll confirm the result in about a minute.
Proceed?
```

Compute the current value as `position.amount + position.unrealizedPnL.pnL`. (Both fields are USD; this works for unleveraged and leveraged positions — see `account-snapshot.md` §3 for why.) Don't lead with units or with the P&L percentage on a close confirmation — those are useful in a status report, but the current dollar value is what matters when the user is deciding whether to liquidate.

Skip confirmation only when the user has already opted into "execute automatically for this session."

### Step 5 — Execute

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

Trade-execution POSTs follow the at-most-once rule in `execution-invariants.md` §3. **2xx with an order ID** → done, move on. **Explicit error** (4xx non-429, or 5xx) → log and stop, don't retry the same payload. **Ambiguous outcome** (timeout, connection reset, no response, parse error) → don't retry; go straight to Step 6 and check `positions[]` / `ordersForOpen[]`. Only **429** is safe to retry.

#### Sizing — stated percentages are CEILINGS

When the user expresses intent as a percentage (*"buy 5% of equity in BTC"*) — common on agent-portfolios, occasional on main accounts — apply the ceilings rule from `execution-invariants.md` §2:

```
EQUITY_ANCHOR = equity from a fresh /pnl read at the start of this trade
amount_usd    = floor(pct × EQUITY_ANCHOR × 100) / 100
```

Send `Amount: amount_usd` exactly. Mirror image for dollar-form intents — *"buy $300 of BTC"* means send `$300`, not `$305` for cleanliness. Step 6 verification surfaces any over-fill (which would be an agent-side bug) and offers a corrective partial close.

### Step 6 — Verify the order landed

A successful POST means the order was **accepted**, not necessarily filled. To verify what actually happened, read `/pnl` and check three landing states:

| State | Where it lives in `clientPortfolio` | Tell the user |
|---|---|---|
| **Filled** | `positions[]` (look for matching `instrumentID`) | "Position opened at $X." (or for closes: "Position closed.") |
| **Pending market open** | `ordersForOpen[]` (matching `instrumentID`) | "Order accepted; will fill when the market opens." Common for stocks placed out of trading hours, weekends, etc. |
| **Failed / unknown** | Neither | "Order didn't land — please re-try or adjust." Surface any error message you got back from the POST. |

If the failure was a **401** (the credential is no longer valid — typically the user revoked their `x-user-key`), don't categorize it as a generic "failed trade" — surface it specifically per `sso-and-session.md` §3 and stop. The single-trade context makes hallucination less catastrophic than in a bulk flow, but the user still needs to know clearly that **no trade was placed** and that their key needs replacing.

#### Read-back timing — wait 60 seconds before reading `/pnl`

The `/pnl` endpoint is **cached for 60 seconds (rolling)** and the cache covers the **entire `clientPortfolio` response** — `positions[]`, `ordersForOpen[]`, `credit`, `unrealizedPnL`, all of it. See `account-snapshot.md` §1 "60-second response cache" for the full mechanic. The practical consequence:

> **Reading `/pnl` immediately after a successful POST returns the *pre-trade* snapshot.** The just-opened position is NOT yet in `positions[]`. If the agent reads early and sees no AAPL, it will *falsely* report the trade as failed — when in fact the trade landed.

Therefore:

```
sleep(60_000)   // after the POST returns 2xx
const pnl = await fetchPnl()
// now categorize against the table above
```

This applies uniformly to opens, closes, partial closes, and limit orders — any trade whose verification depends on reading the post-trade `clientPortfolio` snapshot. The 60 s wait is a **correctness** requirement, not a UX preference.

**The user-facing implication:** tell the user up front, in Step 4 (confirmation), that you'll come back with the verified result in about a minute. Don't send an interim "Order placed; verifying…" message — just stay quiet and send the bottom-line result once verification completes (see Step 7).

### Step 7 — Report back

Tell the user the outcome in their original framing — **dollars and units, not percentages of equity**. (Percentages of equity are only used for agent-portfolios, per Override A in that skill. Main accounts always use absolute dollar amounts.)

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
- **`/trading/info/trade/history` is real-environment only.** If the user asks "what did I trade this week" on a demo-environment key, the endpoint returns `InsufficientPermissions` — surface a specific message rather than a generic auth error.

## Sanity checks

Cross-cutting invariants (covered by `execution-invariants.md`):

- [ ] **For percentage intents** — anchor freeze (§1) applied; `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100` (§2 ceilings); over-fill at verification triggers a corrective partial close.
- [ ] **Open buffer** (§2) — if `(CASH_ANCHOR − requested_amount) / EQUITY_ANCHOR < 0.01`, the trade is shrunk by 1% (`floor(amount × 0.99 × 100) / 100`); for a dollar-form intent, the under-fill is disclosed in the Step 4 confirmation.
- [ ] **At-most-once** (§3) — only 429 retried; 4xx/5xx and ambiguous outcomes reconciled by reading `positions[]`, never by re-firing.
- [ ] **On 401** (§4) — workflow stops; user is told no trade was placed; for mid-session 401, partial state is reported explicitly.

Single-trade-specific:

- [ ] Environment determined from the key, not from the user (`api-conventions.md` "Demo vs. real environments"); same `{env}` segment used for the pre-flight `/pnl` read and the execution POST.
- [ ] Symbol resolved to a verified `instrumentId`; exact-match check on `internalSymbolFull` performed.
- [ ] For opens: Available Cash computed via `account-snapshot.md` §1 and verified ≥ requested amount.
- [ ] For closes: `positionId` looked up from `positions[]`, not assumed; if multiple positions exist on the same instrument, the user picked which one.
- [ ] User confirmed the trade in plain language before POST, **with the ~1-minute verification wait surfaced up front** so the silence isn't a surprise (Step 4); confirmation skipped only if auto-execute is opted in.
- [ ] `Leverage: 1` sent explicitly when not specified.
- [ ] **Post-execution: 60s wait, then `/pnl` re-read and trade categorized as filled / pending / failed.** The wait is mandated by the 60s response cache that covers `positions[]` and `ordersForOpen[]` — reading earlier returns the pre-trade snapshot and produces false "trade didn't land" reports.
- [ ] Outcome reported in the user's original framing (their dollars or their units), with meaningful side effects (e.g. cash impact) noted — sent as a single bottom-line message, not as an interim "verifying…" plus a follow-up.

## References used by this walkthrough

- `api-conventions.md` — auth, casing, demo/real paths, limit-order SL/TP rules, leverage default, optional-parameter handling.
- `id-resolution.md` — symbol → `instrumentId` resolution and conversation-scoped caching.
- `account-snapshot.md` — Available Cash formula and the rest of the PnL response shape.
- `sso-and-session.md` — handling 401s during execution.

For multi-position actions (build a portfolio, rebalance, conditional/triggered rules), don't use this walkthrough — load `bulk-trading.md`, `rebalancing.md`, or `conditional-rules.md` as appropriate.
