# Rebalancing a Portfolio

Reference for the `etoro-trading-assistant` skill. Load this when the user wants to move their portfolio from its current allocation to a new target — explicitly ("rebalance to my model"), implicitly ("swap MSFT for NVDA at the same size"), or on a recurring schedule. Also load it when a single new trade hits a cash shortfall.

The execution borrows directly from `bulk-trading.md`; the new content here is the **diff calculation** and **phase orchestration** (close-phase + 60-second cache wait + open-phase, sharing the 20 req/min rate-limit budget across both).

---

## Account context

This workflow applies to **both regular eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the regular-account default.

> **Agent-portfolio override:** if you reached this reference from the `etoro-agent-portfolios` skill, apply that skill's user-facing-numbers rule — replace dollar amounts with **percentages of equity** in every user-facing message. Internally compute `amount_usd = pct × equity` for the API calls. Endpoint shapes, validation rules, phase ordering, the 60-second cache wait, and the 20 req/min rate limit are all identical.

---

## When to use

- User asks to rebalance to a new target allocation.
- Recurring auto-rebalance ("rebalance weekly to my model portfolio").
- User asks to swap one instrument for another in size, or to scale a position up/down.
- **User asks to open a new position (or a small bulk) and there isn't enough available cash.** Closing or partially-closing existing positions to free cash is part of the same flow — see **Insufficient-cash variant** below.

---

## Insufficient-cash variant

When the trigger is a single new trade (or a small bulk) hitting an out-of-cash situation, the rebalance is narrower than a full target-replacement: only enough existing exposure needs to be reduced to free the required cash; the rest of the portfolio stays put.

### A. Compute the shortfall

```
required_cash = Σ(planned_open_amount_usd)
shortfall = required_cash − available_cash
```

If `shortfall ≤ 0`, this isn't a rebalance — execute the open(s) directly via `single-trade-walkthrough.md` (or `bulk-trading.md` for a multi-position open) without ever entering this reference.

### B. Pick which positions to reduce

Order of preference (ask the user when uncertain):

1. **User-specified.** "Close MSFT to fund this." Always honor an explicit instruction.
2. **Above-target / lowest-conviction.** If a model portfolio or target allocation is in play, reduce positions that are above their target weight first.
3. **Proportional reduction across the existing portfolio.** Least disruptive default — each position contributes `shortfall × (position.amount / total_invested)`. Round per-position units sensibly to avoid sub-unit closes.
4. **Largest position first.** Last-resort fallback when proportional doesn't make sense (e.g. one position dominates the portfolio). Concentrates the cost on a single instrument.

Always **confirm the close plan with the user** in approval mode. Even in auto-rebalance mode, log the chosen reduction strategy.

### C. Execute closes, then opens

From here the flow is identical to a standard rebalance — reuse §§ 5–8 below:

- **Phase 1** (§ 5) — execute the partial closes; rate-limit budget shared across both phases.
- **Wait 60 seconds** for the PnL cache to refresh (§ 6).
- **Verify** the freed cash equals the shortfall (§ 6).
- **Phase 2** (§ 7) — execute the original open(s). If the user only asked for one new trade, Phase 2 is a single POST. The phase ordering and 60s cache wait still apply.
- **Verify final state** (§ 8).

### D. Communicate the trade-off to the user

Be explicit that opening this trade costs them existing exposure. Regular-account / dollar version:

```
To open AAPL at $1,500, I need to free $300 of cash.
I'll partially close MSFT (currently worth $1,200, reducing to $900) to fund this.

Net effect: AAPL +$1,500, MSFT −$300, available cash −$1,200.
Estimated time: ~95 seconds.

Proceed?
```

Agent-portfolio context (percentages):

```
To open AAPL at 5%, I need to free 3% of cash.
I'll partially close MSFT (currently 12% → 9%) to fund this.

Net effect: AAPL +5%, MSFT −3%, available cash −2%.
Estimated time: ~95 seconds.

Proceed?
```

This avoids the surprise of "I asked to open AAPL, why did MSFT shrink?" — the user always sees the cause-and-effect before confirming.

---

## 1. Read current state

```
GET /trading/info/{env}/pnl
```

Compute, per `account-snapshot.md`:

- Equity, Available Cash.
- For each open position, its current dollar value (`position.amount + position.unrealizedPnL.pnL`) and current weight (`position.amount / equity` — useful for diff math even on regular accounts).
- Capture pending orders too (`ordersForOpen[]`, `orders[]`) — they affect available cash but aren't yet filled positions.

---

## 2. Compute the diff against target

The diff math works equally with dollars or weights. Pick the unit that matches the user's target.

For each instrument that appears in either current or target:

```
diff_dollars = target_dollars − current_dollars
   (or)
diff_weight = target_weight − current_weight
```

| Diff | Meaning | Action |
|---|---|---|
| `> +tolerance` | Need to add | Open new position OR add to existing |
| `< −tolerance` | Need to reduce | Partial close OR full close |
| `≈ 0` (within tolerance) | Already aligned | No action |

Use a **tolerance** that matches the unit:

- Dollar tolerance: ~$50 (or 1% of position value, whichever is bigger) — moving smaller amounts wastes a rate-limit slot.
- Weight tolerance: ±0.5%.

Instruments in current but not in target → full close.
Instruments in target but not in current → fresh open.

### Validate the target plan

Apply the same rules as a fresh bulk build — see `bulk-trading.md` §1:

- Dollar plans: `Σ target dollars ≤ equity`.
- Percentage plans: `Σ target weights ≤ 100%`.
- ≤ 20 instruments in the target.
- All target symbols resolve to `instrumentID`.

---

## 3. Split into Phase 1 (closes) and Phase 2 (opens)

### Phase 1 — closes / partial closes

For every instrument with negative diff (need to reduce):

- **If new value = 0** (full exit): full close.
  ```
  POST /trading/execution/{env}/market-close-orders/positions/{positionId}
  Body: { "UnitsToDeduct": null }
  ```
- **If new value > 0 but lower** (partial close): compute units to deduct.
  ```
  units_to_deduct = round(position.units × (1 − new_value / current_value))
  ```
  Where `new_value / current_value` works in either dollars or weights — both ratios are equivalent.
  ```
  POST /trading/execution/{env}/market-close-orders/positions/{positionId}
  Body: { "UnitsToDeduct": <units_to_deduct> }
  ```

If a target instrument is missing from current entirely, it goes to Phase 2 only.

### Phase 2 — opens / increases

For every instrument with positive diff (need to add):

- **If position doesn't currently exist**: standard bulk open with `Amount = diff_dollars`.
- **If position exists and target is higher**: open *another* order with `Amount = diff_dollars`. eToro doesn't have an "increase position" call — you place a second order alongside the existing position. (When summing for display, both will appear in `positions[]` and add together against the target.)

---

## 4. Confirm with the user (approval mode = "ask")

Regular-account version:

```
Rebalance plan:

Closing / reducing:
- AAPL:  $3,000 → $2,000  (partial close: ~33% of position)
- MSFT:  $2,500 → $0       (full close)
- BTC:   $2,000 → $1,500   (partial close: 25% of position)

Opening / increasing:
- TSLA:  $0    → $1,500    (new position)
- NVDA: $1,000 → $2,000    (additional order)

Estimated time: ~95 seconds
  (~25s of close pacing + 60s PnL-cache wait + ~10s of open pacing).

Proceed?
```

For auto-rebalance mode, skip approval but log the plan internally before executing.

---

## 5. Execute Phase 1 (closes)

Use the same execution patterns as `bulk-trading.md` §4:

- Pace ≥ 3 seconds between requests.
- 429 backoff: 15s → 30s → 60s, max 3 retries per trade.
- Continue on individual non-429 failures; capture for the final report.
- **At-most-once delivery applies to closes too** (per `bulk-trading.md` §4 "Response classification"). Only 429 is retried; 4xx/5xx and ambiguous responses (timeout, connection drop) are logged as failed/ambiguous and reconciled at the §6 verification step by reading `positions[]`. Never re-fire a close on ambiguity — a duplicate close on a partial-close payload can over-liquidate the position.
- **On 401: STOP the entire rebalance immediately** (per `bulk-trading.md` and `sso-and-session.md` §§3–4). Don't continue into Phase 2 — all subsequent trades will fail with the same error. Report which closes succeeded and that the rebalance is incomplete; never describe a rebalance as "done" when it isn't.

**The 20 req/min rate-limit budget is shared across opens AND closes** — track total trade-execution requests across both phases of this single rebalance.

---

## 6. Wait 60 seconds, then re-verify cash

The PnL endpoint has a 60-second cache. **Do not trust available-cash numbers until 60s after your last close.** Skipping this wait is the single biggest source of "I tried to open and got `insufficient_funds` even though I just closed enough" bugs.

```
sleep(60_000)
const pnl = await fetchPnl()
const availableCash = computeAvailableCash(pnl)  // see account-snapshot.md §1
```

Confirm `availableCash ≥ Σ(planned_open_amounts)`. If short:

- **Some closes may have failed silently** — verify by checking `positions[]` against the close-plan. Any position that should have been closed but is still in `positions[]` indicates a failed close.
- **Pending close orders** (`orders[]`) may not have settled yet — wait another 60s and re-check.
- **As a last resort**: scale the open-plan proportionally to the actual available cash, OR abort with a clear message to the user explaining the shortfall.

---

## 7. Execute Phase 2 (opens)

Same as `bulk-trading.md` §4, with one constraint: the rate-limit budget is **what's left** of the 20 req/min after Phase 1.

If Phase 1 used 12 trade requests in the last minute, Phase 2 can fire 8 trades immediately, then must wait for the rolling-window to free up. Track this — don't assume a fresh budget after the 60s cache wait (the rate-limit window is request-time, not phase-boundary).

---

## 8. Final verification

- Wait another 60s for cache refresh after the last open.
- Re-read PnL.
- Compare current allocation to target.
- Categorize each instrument as filled / pending market open / failed (see `bulk-trading.md` §5 for the categorization logic).
- Report to the user using the same template as bulk-trading, in their unit (dollars or percentages):

```
Rebalance complete:
- Filled: 5 trades (AAPL, MSFT, BTC partial close; TSLA open; NVDA add).
- Pending market open: 0.
- Failed: 0.

Current allocation matches your target within $40 (or, in agent-portfolio context: 0.3%). The portfolio is rebalanced.
```

---

## 9. Sanity checks

- [ ] Diff is computed against current state (in dollars or weights, matching the user's target unit).
- [ ] Target plan validated like a fresh bulk plan: not mixed-unit, sum ≤ available cash (or ≤ 100%), ≤ 20 instruments, all `instrumentID`s resolved.
- [ ] Phase 1 (closes) completes BEFORE Phase 2 (opens) begins.
- [ ] **At-most-once delivery on every POST in both phases** — only 429 is retried; 4xx/5xx/ambiguous are reconciled by reading `/pnl`, never by re-firing. Especially important on closes: a duplicated partial close can over-liquidate the position.
- [ ] **60-second PnL cache wait between phases** — non-negotiable.
- [ ] Available cash re-verified after closes; missing cash investigated (silent close failure? pending orders?) before proceeding.
- [ ] Rate-limit budget tracked across BOTH phases; no assumption of a fresh 20/min after the cache wait.
- [ ] Final state reported in the user's unit (dollars on regular accounts; percentages in agent-portfolio context); pending-market-open status surfaced.
- [ ] For auto-rebalance: plan logged before execution so a stuck batch can be diagnosed later.
