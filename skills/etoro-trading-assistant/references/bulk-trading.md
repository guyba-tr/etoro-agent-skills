# Bulk Trading: Building or Updating a Multi-Position Portfolio

Reference for the `etoro-trading-assistant` skill. Load this when the user has approved a multi-position plan and the agent is about to execute.

The agent's job here is to execute many opens accurately, respect the 20 req/min trade-execution rate limit, verify the result, and explain the outcome — including any pending-market-open orders — back to the user.

---

## Account context

This workflow applies to **both regular eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the regular-account default.

> **Agent-portfolio override:** if you reached this reference from the `etoro-agent-portfolios` skill, apply that skill's user-facing-numbers rule — replace dollar amounts with **percentages of equity** in every user-facing message (confirmations, reports, error explanations). Internally you still compute `amount_usd = pct × equity` to make the API calls (the API always takes USD); the user just never sees the dollar number. Endpoint shapes, validation rules, rate limits, and timing are all identical.

---

## When to use

- User asks to build a portfolio of multiple instruments in one go ("buy these 8 stocks").
- User has approved a planning step that produced a multi-position allocation.
- Right after Steps 4–6 of the agent-portfolios onboarding flow (in agent-portfolio context).

For an existing portfolio that needs to move to a new allocation, see `rebalancing.md` instead — the close-then-open ordering matters.

---

## 1. Validate the plan

### Input format — dollars or percentages

The user (or planning step) provides the plan in one of two forms:

- **Dollar amounts** (regular-account default): *"$2,500 in AAPL, $1,500 in MSFT, $1,000 in BTC."* Use these directly as `Amount` values in the API calls.
- **Percentages of equity** (agent-portfolio default; some users on regular accounts also prefer this): *"25% AAPL, 15% MSFT, 10% BTC."* Convert internally via `amount_usd = pct × equity`. Preserve the percentage form for user-facing output if they entered it that way.

Reject plans that **mix the two forms** (e.g. "AAPL at 25%, MSFT at $1,500") — ask the user to standardize.

### Sufficiency check

These checks happen against the frozen anchors from §2 (read `/pnl` first if you haven't already; both phases of validation use the same snapshot).

- **For dollar plans:** `Σ planned dollars ≤ CASH_ANCHOR`. If not, reject and surface the gap.
- **For percentage plans:** `Σ percentages ≤ 100%`. If less than 100% and no explicit cash entry, treat the remainder as cash buffer (don't auto-deploy). If more than 100%, reject. Then *also* verify `Σ amount_usd ≤ CASH_ANCHOR` once converted — a percentage plan can pass the 100% check but still fail the cash check if pending orders / mirrors are tying up funds.

### Instrument count

- **Recommend ≤ 20 instruments** per bulk build. Reasons:
  - Rate limit: 20 req/min × 3s spacing = ~60s minimum batch time even before failures.
  - Tracking: per-position weights below ~2.5% (or per-position amounts below ~$50 on a $2,000 portfolio) round oddly when converted to integer units.
  - User comprehension: more than 20 lines of allocation is hard to scan.
- If the user insists on more, comply but warn about execution time.

### Instrument ID resolution

- Resolve **every** symbol → `instrumentID` BEFORE starting execution. Use `/market-data/search?internalSymbolFull=` (see `id-resolution.md`).
- If any symbol fails to resolve, **reject the whole plan**. Don't begin a partial bulk — leaving the user with 17 of 20 positions opened is worse than 0 of 20 with a clear error.

---

## 2. Anchor equity and cash for the workflow

Before sizing or executing anything, fetch the account state and **freeze two anchors** that don't change for the duration of this workflow:

```
pnl = GET /trading/info/{env}/pnl

EQUITY_ANCHOR = equity(pnl)         // unit of user intent: "X% in symbol Y" means X% × EQUITY_ANCHOR
CASH_ANCHOR   = available_cash(pnl) // execution constraint at workflow start
```

Use the formulas in `account-snapshot.md` §1 to compute both. **Neither value is recomputed mid-workflow** — every sizing decision, every cumulative check, and every post-fill verification compares against these frozen numbers.

> **Why freeze.** Equity drifts with market movement on existing positions; cash is stable in single-actor mode (no other client opening or closing positions on the account). If the agent recomputed `pct × current_equity` mid-flow, the same "30% in BTC" intent would resolve to different dollar amounts depending on when the calculation ran — and the post-fill percentage would slide too. Freezing makes percentages deterministic and verifiable.

### Sufficiency check

`total_planned = Σ(amount_usd)` where each `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100` (see §4 Sizing). If `total_planned > CASH_ANCHOR`, **stop — do not start executing, and do not silently switch to a rebalance flow.** Present the gap and let the user choose explicitly. Two situations:

- **Initial build (regular account or agent-portfolio)**: there are no existing positions to close, so the only meaningful response is *"shrink the plan."* Tell the user the gap (in dollars on regular accounts, in percentages on agent-portfolios per the override) and ask them to revise.

- **New trade(s) on an existing portfolio with insufficient cash**: closing existing positions to fund the new trade IS an option, but it requires **explicit user consent** — closes are destructive actions, and the foundational norm in `etoro-trading-assistant`'s SKILL.md is *"Any open / close / cancel / modify requires explicit user confirmation"*. **Don't assume the user wants to liquidate existing exposure to fund the new request.** Present both options and let them pick:

  ```
  You asked me to open positions totalling $8,000, but only $5,000 is available.
  Two options:

    1. Shrink the plan to fit $5,000 — tell me which positions to drop or reduce.
    2. Free up the missing $3,000 by closing or reducing some of your existing
       positions. I'll propose which positions to reduce and confirm with you
       before any closes happen.

  Which would you like?
  ```

  **Only if the user picks option 2** (or if their original request already included an explicit close-to-fund instruction, e.g. *"buy $8,000 of these and close MSFT to cover it"*) do you load `rebalancing.md` and run the "Insufficient-cash variant" — and even then, the close plan is shown for confirmation before Phase 1 begins.

  In agent-portfolio context, frame both options in percentages instead of dollars.

---

## 3. Confirm with the user (if approval mode is "ask")

In approval-required mode, show the proposed allocation and the expected duration. **Match the user's input format** — dollars if they gave dollars, percentages if they gave percentages (or always percentages if running in agent-portfolio context):

```
I'm about to open 8 positions on your real account:
- AAPL:   $2,500
- MSFT:   $1,500
- GOOGL:  $1,000
- BTC:    $2,000
- ETH:    $1,000
- NVDA:     $800
- TSLA:     $700
- (cash):   $500
Total: $10,000

Proceed?
```

If approval mode is "auto" (e.g. recurring rebalance), skip this and log the plan internally.

---

## 4. Execute the bulk opens

### Endpoint

For each instrument, call:

```
POST /trading/execution/{env}/market-open-orders/by-amount
```

```json
{
  "InstrumentID": <resolved_id>,
  "IsBuy": true,
  "Leverage": 1,
  "Amount": <amount_usd>
}
```

(Note `InstrumentID` capital `D` in the request body — see `api-conventions.md` casing notes.)

Always send `Leverage: 1` unless the user explicitly asked for leverage (per `api-conventions.md`).

### Sizing — stated allocations are CEILINGS, not targets

Every percentage and dollar amount the user agreed to is an **upper bound**. The actual filled allocation must be ≤ the stated figure. **Always under-fill rather than over-fill** — a small under-allocation is invisible and recoverable; an over-allocation requires an unwinding partial close, surprises the user, and can incur fees/spread on the corrective trade.

**Three rules together guarantee the ceiling:**

1. **Floor when converting percentage → dollars.** Anchor on `EQUITY_ANCHOR` from §2 (NOT current equity). Floor to cents — never round to a "nice number" like $300 when the math gives $295.40.
  ```
   amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100
  ```
2. **Cumulative check before each send.** Maintain `spent_so_far`. Before firing trade `i`, verify:
  ```
   spent_so_far + amount_usd[i] ≤ Σ planned amount_usd  (= total_planned from §2)
   spent_so_far + amount_usd[i] ≤ CASH_ANCHOR
  ```
   If either constraint would be violated, that's an arithmetic bug in the planner — abort the batch and recompute, do **not** silently shrink-and-fire (the user agreed to a specific plan).
3. **Send the floored dollar amount.** Since eToro has no amount-slippage (`Amount: $X → position.amount = $X` exactly), the floored amount IS the exact filled value. The percentage of `EQUITY_ANCHOR` lands at or just under the stated cap.

For **dollar-form plans** the same discipline applies in mirror image — the user's stated dollar figure is the ceiling. Don't round up "$295" to "$300" for cleanliness; send `$295` exactly.

### Pacing

- **≥ 3 seconds between consecutive trade-execution requests.** This includes opens AND closes during a single batch — the 20 req/min budget is shared.
- Track requests-per-rolling-minute. If you're approaching 20, pause longer.
- Don't parallelize. Bulk-fire is exactly what the rate limiter is designed to reject.

### Response classification — at-most-once delivery

**Trade-execution POSTs are not idempotent.** The eToro API has no idempotency key on `/market-open-orders/`* or `/market-close-orders/*`, so resending the same request after an ambiguous outcome can — and does — produce duplicate positions. Treat every trade request with the **at-most-once** discipline:


| Outcome                                                                | Meaning                                                     | Action                                                                                                             |
| ---------------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **HTTP 200 / 201** with a body containing an `orderId` (or equivalent) | Server accepted and recorded the order                      | **Done. Don't resend. Move on.**                                                                                   |
| **HTTP 4xx** (except 429) — `400`, `403`, `404`, etc.                  | Server rejected the request explicitly; nothing was placed  | Log as failed; safe to surface to the user; **do not retry the same payload** (a 400 won't become a 200)           |
| **HTTP 401**                                                           | Credentials invalid                                         | **STOP the entire batch.** See "Per-trade failure handling" below.                                                 |
| **HTTP 429**                                                           | Rate-limited; server explicitly told you it didn't process  | Safe to retry (the only safe retry case) — see "429 backoff" below                                                 |
| **HTTP 5xx** with a body                                               | Server processed and failed                                 | Log as failed; **do not retry** — you don't know whether the order was placed before the failure                   |
| **Timeout / connection reset / no response / parse error**             | Truly ambiguous — the order may or may not have been placed | **Do not retry.** Mark the trade as ambiguous, continue the batch, and let §5 verification reconcile it via `/pnl` |


**Why "do not retry" on ambiguous and 5xx outcomes:** the request might have reached the server, executed, and the response was lost on the way back. A retry in that case opens a duplicate position — the exact failure mode users hit. A missing position is recoverable (verify, then place it again deliberately); a duplicate is messy to unwind.

The only response that justifies a same-payload retry is **429** (the server explicitly tells you it didn't process the request). Everything else either succeeded, failed cleanly, or is ambiguous — and ambiguous gets resolved at verification time, not by hopeful re-firing.

### 429 backoff

429 is the **one** retry-safe failure mode (the server explicitly states the request wasn't processed):

- First 429: wait **15 seconds**, retry the same trade.
- Second 429 on the same trade: wait **30 seconds**, retry.
- Third: wait **60 seconds**, retry.
- After three failures, mark the trade as failed and continue with the next.

### Per-trade failure handling

- **401 errors: STOP the entire batch immediately.** A 401 means the API credentials are no longer valid — typically the user revoked their `x-user-key` mid-session. Subsequent trades would all fail with the same error. **Do not continue.** Report which trades succeeded BEFORE the 401 explicitly to the user — naming each instrument and amount — and ask them to provide a new key. **Never describe the build as "complete" when it isn't.** See `sso-and-session.md` §§3–4 for the full recovery flow and communication template.
- **Other non-429 errors (400, 5xx) on an individual order:** log, capture the failure, **continue with the remaining trades**. Don't abort the whole batch — partial success is more useful than nothing. **Do not retry these** (per the at-most-once rule above); reconcile their actual state at verification.
- **Ambiguous outcomes (timeout, connection drop):** log as `ambiguous`, continue the batch. Verification in §5 determines whether the order actually landed.
- Capture per-trade outcomes — note the four-state status (not just ok/failed):

```typescript
type TradeStatus =
  | 'ok'         // explicit 2xx with orderId
  | 'failed'     // explicit 4xx/5xx response
  | 'ambiguous'  // no response, timeout, connection reset — actual outcome unknown
  | 'rate_limited_giveup'; // 429 retried 3× and still failed

const results: Array<{
  symbol: string;
  status: TradeStatus;
  error?: string;
}> = [];

for (const trade of plan.trades) {
  await sleep(3000); // pacing
  try {
    const res = await openPosition(trade.instrumentId, trade.amountUsd, trade.isBuy ?? true);
    if (res.orderId) {
      results.push({ symbol: trade.symbol, status: 'ok' });
    } else {
      results.push({ symbol: trade.symbol, status: 'ambiguous', error: 'unexpected response shape' });
    }
  } catch (err) {
    if (err.statusCode === 429) {
      // exponential backoff loop (15s -> 30s -> 60s), max 3 retries
      // on giveup: results.push({ symbol, status: 'rate_limited_giveup' })
    } else if (err.statusCode === 401) {
      // STOP the entire batch — see "Per-trade failure handling" 401 rule above
      throw err;
    } else if (err.statusCode >= 400 && err.statusCode < 600) {
      // explicit error response — log, do NOT retry, continue batch
      results.push({ symbol: trade.symbol, status: 'failed', error: `HTTP ${err.statusCode}: ${err.message}` });
    } else {
      // network error, timeout, connection reset, no response — outcome is ambiguous
      // do NOT retry — verification in §5 will reconcile
      results.push({ symbol: trade.symbol, status: 'ambiguous', error: err.message });
    }
  }
}
```

---

## 5. Verify the orders landed

A successful POST does **not** mean the position is open — it means the order was accepted. Three landing states are possible — and verification is **also** how the at-most-once rule resolves any `ambiguous` trades from §4.

After the last execution, **wait 60 seconds** for the PnL cache to refresh, then:

```
GET /trading/info/{env}/pnl
```

Compare against the plan:


| State                   | Where it lives in `clientPortfolio`          | Meaning                                                                        | Tell the user                           |
| ----------------------- | -------------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------- |
| **Filled**              | `positions[]` (find matching `instrumentID`) | Order executed; user has a real position                                       | "Position opened"                       |
| **Pending market open** | `ordersForOpen[]` (matching `instrumentID`)  | Order accepted but markets are closed (e.g. stocks on weekend or out-of-hours) | "Pending — will fill when market opens" |
| **Failed / unknown**    | Neither array                                | API error, validation failure, or order silently dropped                       | "Failed — please re-review"             |


### Reconciling ambiguous trades

For every trade marked `ambiguous` in §4, check whether it appears in `positions[]` or `ordersForOpen[]`:

- **Found in either array** → the order *did* land despite the ambiguous response. Reclassify as filled or pending. **Do not** place another order — that's the duplicate-prevention point.
- **Not found in either array** → the order didn't land. Now (and only now) is it safe to ask the user whether to retry it as a fresh trade. Surface it as ambiguous-but-missing in the report so the user can decide.

This is the entire point of the at-most-once + post-batch verification pattern: ambiguity is resolved by reading state, not by re-firing requests.

### Over-allocation check (against the frozen anchor)

For every position that landed in `positions[]`, verify the actual fill against the §2 anchor and the §4 sizing rule:

```
expected_amount = floor(stated_pct × EQUITY_ANCHOR × 100) / 100   // or the user's stated dollar figure for dollar plans
actual_amount   = position.amount

if actual_amount > expected_amount:
  // CEILING VIOLATION — the agent's own arithmetic produced an over-allocation.
  // Surface it AND offer a corrective partial close.
```

Since eToro has no amount-slippage, `actual_amount` should equal `expected_amount` exactly. Any over-fill is genuinely an agent-side bug — most often: an inflated equity reference, a round-up bug, or a compounded-base error. **Do not** silently accept an over-allocation.

When detected, present it clearly and offer the fix:

```
⚠ BTC filled at $305 (30.5% of equity). Stated cap was 30% ($300).
This is an over-allocation by $5. Trim by partial-closing $5 of BTC to bring it to exactly 30%? [Y/n]
```

Compute the corrective close as `over = actual_amount − expected_amount`, then partial-close `over` worth of the position (translate to `UnitsToDeduct` per `single-trade-walkthrough.md` Step 6). Apply the at-most-once discipline to the corrective close too — don't retry on ambiguous outcomes; re-verify via `/pnl` instead.

Verify allocation against `EQUITY_ANCHOR`, not against current/post-execution equity. Equity may have moved with market action on other positions during the build — a position's percentage of *current* equity can drift even when its dollar size is exactly correct. The anchor is the user's intent; current equity is the moving market.

### Communication template

In approval mode, after executing (regular-account / dollar version):

```
Build complete. Status:
- Filled: 6 positions (AAPL $2,500; MSFT $1,500; GOOGL $1,000; BTC $2,000; ETH $1,000; SOL $700).
- Pending market open: 2 orders (TSLA $700, NVDA $800 — markets closed; will fill at open).
- Failed: 0.

You've deployed $9,500 of $10,000; the remaining $500 will land once the pending orders fill.
```

In agent-portfolio context, the same report uses percentages:

```
Build complete. Status:
- Filled: 6 positions (AAPL 25%; MSFT 15%; GOOGL 10%; BTC 20%; ETH 10%; SOL 7%).
- Pending market open: 2 orders (TSLA 7%, NVDA 8%).
- Failed: 0.

Your portfolio is ~88% allocated; the remaining 12% will land once the pending orders fill.
```

If anything failed, surface the failures explicitly with the error reason and suggest a follow-up: re-execute, adjust the plan, or skip them.

---

## 6. Sanity checks

- Plan input validated: dollar form OR percentage form, not mixed.
- `**EQUITY_ANCHOR` and `CASH_ANCHOR` frozen at workflow start** (§2) and used for ALL sizing, sufficiency, and verification — never recomputed mid-flow.
- Sufficiency check passed: `Σ amount_usd ≤ CASH_ANCHOR` (and for percentages, also `Σ ≤ 100%`).
- ≤ 20 instruments recommended (warn the user if more).
- All `instrumentID`s resolved up front; partial-resolution plans rejected entirely.
- **Sizing — stated allocations are CEILINGS**: `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100`; never round up; never round to "nice numbers"; cumulative `spent_so_far + next_amount ≤ total_planned` checked before each send.
- Trade-execution requests spaced ≥3s; 429 handled with 15s → 30s → 60s backoff; per-trade limit of 3 retries.
- **At-most-once delivery enforced**: only 429 triggers a same-payload retry. 4xx (except 429), 5xx, and ambiguous outcomes (timeout / connection drop / no response) are never retried — they're reconciled at verification time via `/pnl`. This is the duplicate-position prevention rule.
- **On 401: stop the entire batch immediately, report which trades succeeded before the failure, ask for a new credential.** Never report the batch as "complete" when it isn't.
- Other per-trade failures (400, 5xx) are logged and reported but don't abort the batch and are never retried.
- Post-execution: 60s wait, then `/pnl` re-read; results categorized as filled / pending / failed / ambiguous-but-missing against the plan; ambiguous trades resolved by checking `positions[]`/`ordersForOpen[]`, never by re-firing.
- **Over-allocation check** (§5): for each filled position, `actual_amount ≤ floor(stated_pct × EQUITY_ANCHOR × 100) / 100`. Any over-fill is flagged and a corrective partial close is offered. Allocation is verified against `EQUITY_ANCHOR`, NOT current/post-execution equity.
- User-facing report uses **the same unit the user gave you** (dollars on regular accounts; percentages in agent-portfolio context); pending-market-open distinction explicitly surfaced; percentages reported are computed against `EQUITY_ANCHOR`, not current equity.

