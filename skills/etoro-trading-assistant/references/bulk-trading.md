# Bulk Trading: Building or Updating a Multi-Position Portfolio

Reference for the `etoro-trading-assistant` skill. Load this when the user has approved a multi-position plan and the agent is about to execute.

The agent's job here is to execute many opens accurately, respect the 20 req/min trade-execution rate limit, verify the result, and explain the outcome — including any pending-market-open orders — back to the user.

---

## Account context

This workflow applies to **both main eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the main-account default.

> **Agent-portfolio override:** if you reached this reference from the `etoro-agent-portfolios` skill, apply **Override A** from that skill — replace dollar amounts with **percentages of equity** in every user-facing message (confirmations, reports, error explanations). Internally you still compute `amount_usd = pct × EQUITY_ANCHOR` to make the API calls (the API always takes USD); the user just never sees the dollar number. Endpoint shapes, validation rules, rate limits, and timing are all identical.

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

- **Dollar amounts** (main-account default): *"$2,500 in AAPL, $1,500 in MSFT, $1,000 in BTC."* Use these directly as `Amount` values in the API calls.
- **Percentages of equity** (agent-portfolio default; some users on main accounts also prefer this): *"25% AAPL, 15% MSFT, 10% BTC."* Convert internally via `amount_usd = pct × equity`. Preserve the percentage form for user-facing output if they entered it that way.

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

Before sizing or executing anything, freeze `EQUITY_ANCHOR` and `CASH_ANCHOR` from a fresh `/pnl` read — see **`execution-invariants.md` §1** for the canonical anchor-freeze rule and rationale.

Both values are used for ALL sizing decisions, the cumulative `spent_so_far` check (§4), and post-execution verification (§5). Neither is recomputed mid-workflow.

### Sufficiency check

`total_planned = Σ(amount_usd)` where each `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100` (see §4 Sizing). If `total_planned > CASH_ANCHOR`, **stop — do not start executing, and do not silently switch to a rebalance flow.** Present the gap and let the user choose explicitly. Two situations:

- **Initial build (main account or agent-portfolio)**: there are no existing positions to close, so the only meaningful response is *"shrink the plan."* Tell the user the gap (in dollars on main accounts, in percentages on agent-portfolios per the override) and ask them to revise.

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
I'm about to open 8 positions:
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

Apply `execution-invariants.md` §2 (the canonical ceilings rule) to every trade in the batch. Bulk-specific application:

1. **Floor when converting**: `amount_usd[i] = floor(pct_i × EQUITY_ANCHOR × 100) / 100`. Never round up; never round to "nice numbers" like $300 when math gives $295.40.
2. **Cumulative check before each send.** Maintain `spent_so_far`. Before firing trade `i`:
   ```
   spent_so_far + amount_usd[i] ≤ Σ planned amount_usd  (= total_planned from §2)
   spent_so_far + amount_usd[i] ≤ CASH_ANCHOR
   ```
   If violated, abort the batch and recompute — don't silently shrink-and-fire (the user agreed to a specific plan).
3. **Open buffer if planned cash < 1% of equity** (per `execution-invariants.md` §2 "Open buffer"). After the floor in step 1, before any POST:
   ```
   total_planned    = Σ amount_usd[i]
   planned_cash_pct = (CASH_ANCHOR − total_planned) / EQUITY_ANCHOR

   if planned_cash_pct < 0.01:
     for each i: amount_usd[i] = floor(amount_usd[i] × 0.99 × 100) / 100
   ```
   For percentage-form bulk plans, this re-floor is silent — the user-facing percentages don't change (the under-fill is invisible at percentage resolution). For dollar-form plans, the per-position dollar amounts change visibly; mention the under-fill in the §3 confirmation message before sending.
4. **Send the floored amount exactly.** Since eToro has no amount-slippage, the floored amount IS the filled value. Mirror image for dollar-form plans — *"$295"* means send `$295`, not `$300` for cleanliness.

### Pacing

- **≥ 3 seconds between consecutive trade-execution requests.** This includes opens AND closes during a single batch — the 20 req/min budget is shared.
- Track requests-per-rolling-minute. If you're approaching 20, pause longer.
- Don't parallelize. Bulk-fire is exactly what the rate limiter is designed to reject.

### Response classification — at-most-once delivery

Every trade in the batch follows the at-most-once rule from `execution-invariants.md` §3 — only 429 triggers a same-payload retry (cadence: 15s → 30s → 60s, max 3); 4xx (non-429), 5xx, and ambiguous outcomes (timeout / connection drop / no response) are logged and reconciled at the §5 verification step, never re-fired.

Bulk-specific application:

- **On 401: STOP the entire batch immediately** (per invariants §4). Subsequent trades would all fail with the same error. Report which trades succeeded BEFORE the 401 explicitly — naming each instrument and amount — and ask the user for a new credential. Never describe the build as "complete" when it isn't. See `sso-and-session.md` §§3–4.
- **Other non-429 errors (400, 5xx) on an individual order:** log, capture the failure, **continue with the remaining trades**. Don't abort the whole batch — partial success is more useful than nothing.
- **Ambiguous outcomes (timeout, connection drop):** log as `ambiguous`, continue the batch. Verification in §5 determines whether the order actually landed.
- Capture per-trade outcomes — use the four-state status from invariants §3:

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

After the last execution, **wait 10 seconds** for the PnL cache to refresh (`account-snapshot.md` §1 "10-second response cache"), then:

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

For every position that landed in `positions[]`, verify per `execution-invariants.md` §2 ("Verify against the anchor, not against current equity"):

```
expected_amount = floor(stated_pct × EQUITY_ANCHOR × 100) / 100   // or the user's stated dollar figure
actual_amount   = position.amount
```

Any `actual_amount > expected_amount` is a ceiling violation (agent-side bug). Surface it clearly and offer a corrective partial close — `over = actual_amount − expected_amount`, then partial-close `over` worth of the position (translate to `UnitsToDeduct` per `single-trade-walkthrough.md` Step 5). The corrective close itself follows at-most-once (invariants §3) — re-verify via `/pnl`, don't retry on ambiguity.

### Communication template

In approval mode, after executing (main-account / dollar version):

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

Cross-cutting invariants (covered by `execution-invariants.md`):

- [ ] **Anchor freeze** — `EQUITY_ANCHOR` / `CASH_ANCHOR` frozen at workflow start and used for ALL sizing, sufficiency, and verification.
- [ ] **Ceilings** — `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100`; cumulative `spent_so_far + next ≤ total_planned ≤ CASH_ANCHOR` checked before each send; post-fill verification against `EQUITY_ANCHOR`, not current equity; over-fills surfaced and corrected via partial close.
- [ ] **Open buffer** — if planned cash < 1% of `EQUITY_ANCHOR`, each `amount_usd[i]` re-floored to `× 0.99` to absorb per-trade fees; disclosed for dollar-form plans, silent for percentage-form plans.
- [ ] **At-most-once** — only 429 retried (15s → 30s → 60s, max 3); 4xx (non-429), 5xx, and ambiguous outcomes reconciled at verification time via `/pnl`, never re-fired.
- [ ] **On 401** — entire batch stops immediately; user is told which trades succeeded and which never executed; no "complete" summary on a partial run.

Bulk-specific:

- [ ] Plan input validated: dollar form OR percentage form, not mixed.
- [ ] Sufficiency check passed: `Σ amount_usd ≤ CASH_ANCHOR` (and for percentages, also `Σ ≤ 100%`).
- [ ] ≤ 20 instruments recommended (warn the user if more).
- [ ] All `instrumentID`s resolved up front; partial-resolution plans rejected entirely.
- [ ] Trade-execution requests spaced ≥ 3s; 20 req/min budget tracked.
- [ ] Other per-trade failures (400, 5xx) logged and reported but don't abort the batch.
- [ ] Post-execution: 10s wait, then `/pnl` re-read; results categorized as filled / pending / failed / ambiguous-but-missing; ambiguous resolved by reading `positions[]`/`ordersForOpen[]`.
- [ ] User-facing report uses the same unit the user gave you (dollars on main accounts; percentages in agent-portfolio context); pending-market-open distinction explicitly surfaced.

