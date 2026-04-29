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
- **User has explicitly approved closing existing positions to fund a new trade that doesn't fit current cash.** This is the "Insufficient-cash variant" below. The trigger is *user consent*, not the cash shortfall itself — when a new trade exceeds available cash, the agent's first move is to **stop and ask** whether to shrink the plan or free cash via closes (see `bulk-trading.md` §2 "Sufficiency check" and `single-trade-walkthrough.md` Step 4 for the consent template). Only after the user picks the close-to-fund path do you enter this reference.

---

## Insufficient-cash variant

**Entry condition: the user has just explicitly approved closing existing positions to fund a new trade.** This is not entered automatically when a new trade exceeds available cash — the consent gate lives in `bulk-trading.md` §2 / `single-trade-walkthrough.md` Step 4. Don't reach this section by silently inferring intent from a cash shortfall.

Once the user has approved the close-to-fund path, the rebalance is narrower than a full target-replacement: only enough existing exposure needs to be reduced to free the required cash; the rest of the portfolio stays put.

### A. Compute the shortfall — and the close target with a buffer

Anchor first per §1, then:

```
required_cash    = Σ(planned_open_amount_usd)   // each amount = floor(pct × EQUITY_ANCHOR × 100) / 100
shortfall        = required_cash − CASH_ANCHOR
close_buffer     = ceil(shortfall × 0.01 × 100) / 100   // 1% of shortfall, rounded UP to cents
close_target     = shortfall + close_buffer
```

**Why the buffer.** The buffer is a **floor on the close** (i.e. close *at least* this much, slightly over the strict shortfall) — the mirror image of the ceiling-on-opens rule in `bulk-trading.md` §4. Without it, a tiny mismatch — partial-close `UnitsToDeduct` rounding to whole units, or any post-close fees nibbling the freed cash — can leave you 0.X% short of the target. That would force a corrective second close-then-wait-60s round, doubling the rebalance time and giving the user a confusing flow. Closing 1% of the shortfall over what's strictly needed eliminates that failure mode while leaving negligible cash idle.

**Direction of rounding:** the buffer is rounded UP (`ceil`), and the per-position close amounts must sum to ≥ `close_target` (also rounded such that we never close *less* than required). This is intentional: in the close phase we err in the direction of more freed cash, opposite to the open phase where we err in the direction of less spent cash.

If `shortfall ≤ 0`, this isn't a rebalance — execute the open(s) directly via `single-trade-walkthrough.md` (or `bulk-trading.md` for a multi-position open) without ever entering this reference.

### B. Pick which positions to reduce

Pick positions to free **at least `close_target`** (= `shortfall + close_buffer`) — never less. Order of preference (ask the user when uncertain):

1. **User-specified.** "Close MSFT to fund this." Always honor an explicit instruction.
2. **Above-target / lowest-conviction.** If a model portfolio or target allocation is in play, reduce positions that are above their target weight first.
3. **Proportional reduction across the existing portfolio.** Least disruptive default — each position contributes `close_target × (position.amount / total_invested)`. Round per-position `UnitsToDeduct` such that the *sum of freed cash is ≥ `close_target`* (round closes UP, never down — under-closing leaves you short of the target and forces another rebalance round).
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

Be explicit that opening this trade costs them existing exposure. Mention the small over-close buffer so the user isn't surprised by a few extra dollars / basis points being freed. Regular-account / dollar version:

```
To open AAPL at $1,500, I need to free $300 of cash (plus a $3 safety buffer
so we don't land short and have to rebalance again — total close target $303).
I'll partially close MSFT (currently worth $1,200, reducing to ~$897) to fund this.

Net effect: AAPL +$1,500, MSFT −$303, available cash −$1,197.
Estimated time: ~95 seconds.

Proceed?
```

Agent-portfolio context (percentages):

```
To open AAPL at 5%, I need to free 3% of cash (plus a small safety buffer of
~0.03% so we don't land short and have to rebalance again).
I'll partially close MSFT (currently 12% → ~8.97%) to fund this.

Net effect: AAPL +5%, MSFT −3.03%, available cash ~−1.97%.
Estimated time: ~95 seconds.

Proceed?
```

This avoids the surprise of "I asked to open AAPL, why did MSFT shrink?" — the user always sees the cause-and-effect before confirming, including the small extra freed by the buffer.

---

## 1. Anchor equity and cash, read current state

```
pnl = GET /trading/info/{env}/pnl
```

**Freeze two anchors that don't change for the duration of this rebalance** (per `bulk-trading.md` §2):

```
EQUITY_ANCHOR = equity(pnl)         // unit of user intent — "rebalance to 30% in BTC" means 30% × EQUITY_ANCHOR
CASH_ANCHOR   = available_cash(pnl) // execution constraint at workflow start
```

Use these for ALL sizing decisions in BOTH phases (closes and opens). Don't re-anchor between phases — the 60s PnL-cache wait happens with the anchors still locked; only the **live cash check** is re-read after the cache refresh, never the anchor.

Also compute, per `account-snapshot.md`:

- For each open position, its current dollar value (`position.amount + position.unrealizedPnL.pnL`) and current weight (`position.amount / EQUITY_ANCHOR` — useful for diff math even on regular accounts).
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

For every instrument with positive diff (need to add) — apply `bulk-trading.md` §4 "Sizing — stated allocations are CEILINGS":

```
amount_usd = floor(target_pct × EQUITY_ANCHOR × 100) / 100   // for percentage targets
           // or just floor(diff_dollars × 100) / 100 for explicit dollar targets
```

- **If position doesn't currently exist**: standard bulk open with `Amount = amount_usd`.
- **If position exists and target is higher**: open *another* order with `Amount = floor(diff_dollars × 100) / 100`. eToro doesn't have an "increase position" call — you place a second order alongside the existing position. (When summing for display, both will appear in `positions[]` and add together against the target.) The combined `position.amount` total must still satisfy the ceiling: `Σ position.amount on this instrument ≤ floor(target_pct × EQUITY_ANCHOR × 100) / 100`.

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
const liveCash = computeAvailableCash(pnl)  // live read, NOT the frozen CASH_ANCHOR
```

Confirm `liveCash ≥ Σ(planned_open_amounts)` (the buffer from §A "Insufficient-cash variant" should make this comfortably true — `liveCash` should land at roughly `required_cash + close_buffer`, never short). If short anyway:

- **Some closes may have failed silently** — verify by checking `positions[]` against the close-plan. Any position that should have been closed but is still in `positions[]` indicates a failed close.
- **Pending close orders** (`orders[]`) may not have settled yet — wait another 60s and re-check.
- **As a last resort**: scale the open-plan proportionally to the actual available cash, OR abort with a clear message to the user explaining the shortfall.

> Note: `liveCash` here is a **live read for execution-readiness**, not a new anchor. `EQUITY_ANCHOR` and `CASH_ANCHOR` from §1 remain frozen — sizing in Phase 2 still uses `EQUITY_ANCHOR`, not the post-close live equity.

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

- [ ] **`EQUITY_ANCHOR` and `CASH_ANCHOR` frozen at workflow start** (§1) and used for sizing in BOTH phases. The 60s wait re-reads cash for execution-readiness only — it doesn't re-anchor.
- [ ] Diff is computed against current state (in dollars or weights, matching the user's target unit).
- [ ] Target plan validated like a fresh bulk plan: not mixed-unit, sum ≤ `CASH_ANCHOR` (or ≤ 100%), ≤ 20 instruments, all `instrumentID`s resolved.
- [ ] **Insufficient-cash variant: `close_target = shortfall + 1% of shortfall (rounded up)`** — closes are over-shot by this small buffer to avoid landing short and triggering a corrective second round.
- [ ] **Closes round UP** (free at least `close_target`, never less) — opposite direction from opens which floor.
- [ ] **Opens round DOWN — stated allocations are CEILINGS**: `amount_usd = floor(target_pct × EQUITY_ANCHOR × 100) / 100`. Any over-fill is flagged at verification and a corrective partial close is offered.
- [ ] Phase 1 (closes) completes BEFORE Phase 2 (opens) begins.
- [ ] **At-most-once delivery on every POST in both phases** — only 429 is retried; 4xx/5xx/ambiguous are reconciled by reading `/pnl`, never by re-firing. Especially important on closes: a duplicated partial close can over-liquidate the position.
- [ ] **60-second PnL cache wait between phases** — non-negotiable.
- [ ] Live cash re-verified after closes (`liveCash ≥ Σ planned_opens`); the buffer should make this comfortable — if short anyway, investigate (silent close failure? pending orders?) before proceeding.
- [ ] Rate-limit budget tracked across BOTH phases; no assumption of a fresh 20/min after the cache wait.
- [ ] Final state reported in the user's unit (dollars on regular accounts; percentages in agent-portfolio context); pending-market-open status surfaced.
- [ ] For auto-rebalance: plan logged before execution so a stuck batch can be diagnosed later.
