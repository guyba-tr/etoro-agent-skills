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

- **For dollar plans:** `Σ planned dollars ≤ available cash`. If not, reject and surface the gap.
- **For percentage plans:** `Σ percentages ≤ 100%`. If less than 100% and no explicit cash entry, treat the remainder as cash buffer (don't auto-deploy). If more than 100%, reject. Then *also* verify `Σ amount_usd ≤ available cash` once converted — a percentage plan can pass the 100% check but still fail the cash check if pending orders / mirrors are tying up funds.

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

## 2. Pre-flight: check equity and cash

Before any open, fetch the current account state:

```
GET /trading/info/{env}/pnl
```

Compute Available Cash and Equity using the formulas in `account-snapshot.md` §1.

`total_planned = Σ(amount_usd)`. If `total_planned > available_cash`:

- **Initial build on a regular account**: tell the user the gap in dollars (e.g. *"You're trying to deploy $8,000 but only $5,000 is available."*) and ask them to revise.
- **Initial build on an agent-portfolio**: same constraint, expressed in percentages per the agent-portfolios override.
- **A new trade hitting cash shortfall on an existing portfolio**: that's a rebalance trigger — load `rebalancing.md` and use the "Insufficient-cash variant."

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

Estimated execution time: ~25 seconds (rate-limit pacing).

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

### Pacing

- **≥ 3 seconds between consecutive trade-execution requests.** This includes opens AND closes during a single batch — the 20 req/min budget is shared.
- Track requests-per-rolling-minute. If you're approaching 20, pause longer.
- Don't parallelize. Bulk-fire is exactly what the rate limiter is designed to reject.

### 429 backoff

- First 429: wait **15 seconds**, retry the same trade.
- Second 429 on the same trade: wait **30 seconds**, retry.
- Third: wait **60 seconds**, retry.
- After three failures, mark the trade as failed and continue with the next.

### Per-trade failure handling

- **401 errors: STOP the entire batch immediately.** A 401 means the API credentials are no longer valid — typically the user revoked their `x-user-key` mid-session. Subsequent trades would all fail with the same error. **Do not continue.** Report which trades succeeded BEFORE the 401 explicitly to the user — naming each instrument and amount — and ask them to provide a new key. **Never describe the build as "complete" when it isn't.** See `sso-and-session.md` §§3–4 for the full recovery flow and communication template.
- **Other non-429 errors (400, 500, etc.) on an individual order:** log, capture the failure, **continue with the remaining trades**. Don't abort the whole batch — partial success is more useful than nothing. (These are per-trade problems — bad parameters, transient server errors — not credential failures, so subsequent trades may still succeed.)
- Capture per-trade outcomes:

```typescript
const results: Array<{
  symbol: string;
  status: 'ok' | 'failed';
  error?: string;
}> = [];

for (const trade of plan.trades) {
  await sleep(3000); // pacing
  try {
    await openPosition(trade.instrumentId, trade.amountUsd, trade.isBuy ?? true);
    results.push({ symbol: trade.symbol, status: 'ok' });
  } catch (err) {
    if (err.statusCode === 429) {
      // exponential backoff loop, max 3 retries
    } else {
      results.push({ symbol: trade.symbol, status: 'failed', error: err.message });
    }
  }
}
```

---

## 5. Verify the orders landed

A successful POST does **not** mean the position is open — it means the order was accepted. Three landing states are possible.

After the last execution, **wait 60 seconds** for the PnL cache to refresh, then:

```
GET /trading/info/{env}/pnl
```

Compare against the plan:

| State | Where it lives in `clientPortfolio` | Meaning | Tell the user |
|---|---|---|---|
| **Filled** | `positions[]` (find matching `instrumentID`) | Order executed; user has a real position | "Position opened" |
| **Pending market open** | `ordersForOpen[]` (matching `instrumentID`) | Order accepted but markets are closed (e.g. stocks on weekend or out-of-hours) | "Pending — will fill when market opens" |
| **Failed / unknown** | Neither array | API error, validation failure, or order silently dropped | "Failed — please re-review" |

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

- [ ] Plan input validated: dollar form OR percentage form, not mixed.
- [ ] Sufficiency check passed: `Σ dollars ≤ available cash` (and for percentages, also `Σ ≤ 100%`).
- [ ] ≤ 20 instruments recommended (warn the user if more).
- [ ] All `instrumentID`s resolved up front; partial-resolution plans rejected entirely.
- [ ] Trade-execution requests spaced ≥3s; 429 handled with 15s → 30s → 60s backoff; per-trade limit of 3 retries.
- [ ] **On 401: stop the entire batch immediately, report which trades succeeded before the failure, ask for a new credential.** Never report the batch as "complete" when it isn't.
- [ ] Other per-trade failures (400, 500) are logged and reported but don't abort the batch.
- [ ] Post-execution: 60s wait, then `/pnl` re-read; results categorized as filled / pending / failed against the plan.
- [ ] User-facing report uses **the same unit the user gave you** (dollars on regular accounts; percentages in agent-portfolio context); pending-market-open distinction explicitly surfaced.
