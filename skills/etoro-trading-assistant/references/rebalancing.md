# Rebalancing a Portfolio

Reference for the `etoro-trading-assistant` skill. Load this when the user wants to move their portfolio from its current allocation to a new target — explicitly ("rebalance to my model"), implicitly ("swap MSFT for NVDA at the same size"), or on a recurring schedule. Also load it when a single new trade hits a cash shortfall.

The execution borrows directly from `bulk-trading.md`; the new content here is the **diff calculation** and **phase orchestration** (close-phase + 60-second cache wait + open-phase, sharing the 20 req/min rate-limit budget across both).

---

## Account context

This workflow applies to **both main eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the main-account default.

> **Agent-portfolio override:** if you reached this reference from the `etoro-agent-portfolios` skill, apply **Override A** from that skill — replace dollar amounts with **percentages of equity** in every user-facing message. Internally compute `amount_usd = pct × EQUITY_ANCHOR` for the API calls. Endpoint shapes, validation rules, phase ordering, the 60-second cache wait, and the 20 req/min rate limit are all identical.

---

## When to use

- User asks to rebalance to a new target allocation.
- Recurring auto-rebalance ("rebalance weekly to my model portfolio").
- User asks to swap one instrument for another in size, or to scale a position up/down.
- **User has explicitly approved closing existing positions to fund a new trade that doesn't fit current cash.** This is the "Insufficient-cash variant" below. The trigger is *user consent*, not the cash shortfall itself — when a new trade exceeds available cash, the agent's first move is to **stop and ask** whether to shrink the plan or free cash via closes (see `bulk-trading.md` §2 "Sufficiency check" and `single-trade-walkthrough.md` Step 3 for the consent template). Only after the user picks the close-to-fund path do you enter this reference.

---

## Insufficient-cash variant

**Entry condition: the user has just explicitly approved closing existing positions to fund a new trade.** This is not entered automatically when a new trade exceeds available cash — the consent gate lives in `bulk-trading.md` §2 / `single-trade-walkthrough.md` Step 3. Don't reach this section by silently inferring intent from a cash shortfall.

Once the user has approved the close-to-fund path, the rebalance is narrower than a full target-replacement: only enough existing exposure needs to be reduced to free the required cash; the rest of the portfolio stays put.

### A. Compute the shortfall — and the close target with a buffer

Anchor first per §1, then:

```
required_cash    = Σ(planned_open_amount_usd)   // each amount = floor(pct × EQUITY_ANCHOR × 100) / 100
shortfall        = required_cash − CASH_ANCHOR
close_buffer     = ceil(shortfall × 0.01 × 100) / 100   // 1% of shortfall, rounded UP to cents
close_target     = shortfall + close_buffer
```

**Why the buffer.** This is the mirror image of the ceiling-on-opens rule (`execution-invariants.md` §2): closes round **UP**, opens round **DOWN**. Without the buffer, a tiny mismatch — partial-close `UnitsToDeduct` rounding to whole units, or post-close fees nibbling the freed cash — can leave you 0.X% short and force a corrective second close-then-wait-60s round. Closing 1% of the shortfall over what's strictly needed eliminates that failure mode while leaving negligible cash idle.

**Direction of rounding:** the buffer is rounded UP (`ceil`), and per-position close amounts must sum to ≥ `close_target`. In the close phase we err toward more freed cash; in the open phase we err toward less spent cash.

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

Be explicit that opening this trade costs them existing exposure. The over-close buffer can be alluded to in plain language ("a small extra to make sure we don't land short") but **never named as "buffer" with mechanics attached**. Main-account / dollar version:

```
To open AAPL at $1,500, I need to free about $303 of cash (a small extra
beyond $300 so we don't land short).
I'll partially close MSFT (currently worth $1,200, reducing to ~$897) to fund this.

Net effect: AAPL +$1,500, MSFT −$303.
Estimated time: about 2½ minutes.

Proceed?
```

Agent-portfolio context (percentages — Override A: dollars are NEVER customer-facing):

```
To open AAPL at 5%, I need to free about 3% of cash (a small extra beyond
3% so we don't land short).
I'll partially close MSFT (currently 12% → ~9%) to fund this.

Net effect: AAPL +5%, MSFT −3%.
Estimated time: about 2½ minutes.

Proceed?
```

This avoids the surprise of "I asked to open AAPL, why did MSFT shrink?" — the user always sees the cause-and-effect before confirming. **The estimated time is one customer-friendly number; never broken down into "two 60s cache waits", "Phase 1 / Phase 2", or any other mechanism explanation** (per `etoro-trading-assistant/SKILL.md` "Talk in bottom lines, not mechanics").

---

## 1. Anchor equity and cash, read current state

Freeze `EQUITY_ANCHOR` and `CASH_ANCHOR` from a fresh `/pnl` read — see **`execution-invariants.md` §1** for the canonical anchor-freeze rule.

These anchors stay frozen across BOTH phases of the rebalance (closes and opens). The 60s PnL-cache wait between phases re-reads cash for **execution-readiness only** (a live check that closes settled), but never re-anchors. Sizing in Phase 2 still uses `EQUITY_ANCHOR`.

Also compute, per `account-snapshot.md`:

- For each open position, its current dollar value (`position.amount + position.unrealizedPnL.pnL`) and current weight (`position.amount / EQUITY_ANCHOR` — useful for diff math even on main accounts).
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

For every instrument with positive diff (need to add) — apply `execution-invariants.md` §2 (ceilings):

```
amount_usd = floor(target_pct × EQUITY_ANCHOR × 100) / 100   // for percentage targets
           // or just floor(diff_dollars × 100) / 100 for explicit dollar targets
```

- **If position doesn't currently exist**: standard bulk open with `Amount = amount_usd`.
- **If position exists and target is higher**: open *another* order with `Amount = floor(diff_dollars × 100) / 100`. eToro doesn't have an "increase position" call — place a second order alongside the existing one. (Both will appear in `positions[]` and add together against the target.) The combined `position.amount` total must still satisfy the ceiling: `Σ position.amount on this instrument ≤ floor(target_pct × EQUITY_ANCHOR × 100) / 100`.

#### Open buffer if planned post-rebalance cash < 1% of equity

After computing per-position open amounts, apply the open-buffer check from `execution-invariants.md` §2:

```
total_opens               = Σ amount_usd[i]   // Phase 2 opens
post_rebalance_cash_pct   = (CASH_ANCHOR + freed_from_closes − total_opens) / EQUITY_ANCHOR
   // For the insufficient-cash variant, freed_from_closes = close_target ≈ shortfall + close_buffer,
   // so post_rebalance_cash_pct ≈ close_buffer / EQUITY_ANCHOR ≈ 1% of shortfall / EQUITY — usually tiny.

if post_rebalance_cash_pct < 0.01:
  for each i: amount_usd[i] = floor(amount_usd[i] × 0.99 × 100) / 100
```

**Interaction with `close_buffer` in the insufficient-cash variant.** Both buffers stay — they protect against different leakage sources (close-side fees → `close_buffer`; open-side fees → open buffer). On a typical insufficient-cash rebalance, both will be applied: the close phase frees `shortfall + close_buffer`, then Phase 2 opens are each shrunk by 1% if planned post-rebalance cash is < 1% of equity (which it usually is for an insufficient-cash variant by design).

**Disclosure**: percentage-form rebalance plans apply the buffer silently (the user-facing percentages don't change). Dollar-form rebalance plans should mention the per-position under-fill in the §4 plan confirmation, same as the bulk-trading dollar-form rule.

---

## 4. Confirm with the user (approval mode = "ask")

Main-account version:

```
Rebalance plan:

Closing / reducing:
- AAPL:  $3,000 → $2,000  (partial close: ~33% of position)
- MSFT:  $2,500 → $0       (full close)
- BTC:   $2,000 → $1,500   (partial close: 25% of position)

Opening / increasing:
- TSLA:  $0    → $1,500    (new position)
- NVDA: $1,000 → $2,000    (additional order)

Estimated time: about 2½ minutes.

Proceed?
```

The estimated time is **one customer-friendly number**. **Never** break it down for the user as "~25s of close pacing + 60s PnL-cache wait + ~10s of open pacing + 60s wait before final verification" or similar — that's mechanism-narration, banned per `etoro-trading-assistant/SKILL.md` "Talk in bottom lines, not mechanics". The internal planner uses the breakdown to compute the ~2½-minute total; the user only sees the total.

For agent-portfolio context, swap dollars for percentages of equity (Override A):

```
Rebalance plan:

Closing / reducing:
- AAPL:  30% → 20%    (partial close)
- MSFT:  25% → 0%     (full close)
- BTC:   20% → 15%    (partial close)

Opening / increasing:
- TSLA:  0%  → 15%    (new position)
- NVDA: 10%  → 20%    (increase)

Estimated time: about 2½ minutes.

Proceed?
```

For auto-rebalance mode, skip approval but log the plan internally before executing.

---

## 5. Execute Phase 1 (closes)

Use the same execution patterns as `bulk-trading.md` §4:

- Pace ≥ 3 seconds between requests.
- All POSTs follow `execution-invariants.md` §3 (at-most-once) — only 429 retried (15s → 30s → 60s, max 3); 4xx/5xx/ambiguous are reconciled at §6 verification by reading `positions[]`. **Never re-fire a close on ambiguity** — a duplicate partial close can over-liquidate the position.
- **On 401: STOP the entire rebalance immediately** per invariants §4. Don't continue into Phase 2; report which closes succeeded; never describe a rebalance as "done" when it isn't.
- Continue on individual non-429 failures; capture for the final report.

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

Cross-cutting invariants (covered by `execution-invariants.md`):

- [ ] **Anchor freeze** — `EQUITY_ANCHOR` / `CASH_ANCHOR` frozen at workflow start (§1) and used for sizing in BOTH phases; the 60s wait re-reads cash for execution-readiness only.
- [ ] **Ceilings on opens** — `amount_usd = floor(target_pct × EQUITY_ANCHOR × 100) / 100`; over-fills surfaced and corrected.
- [ ] **Open buffer** — if `(CASH_ANCHOR + freed_from_closes − Σ opens) / EQUITY_ANCHOR < 0.01`, each open re-floored to `× 0.99` so per-trade fees don't push displayed cash negative. On the insufficient-cash variant this is almost always triggered (and stacks with the existing close_buffer; both are kept).
- [ ] **At-most-once on every POST in BOTH phases** — only 429 retried; 4xx/5xx/ambiguous reconciled via `/pnl`, never re-fired. Critical on closes: a duplicated partial close can over-liquidate.
- [ ] **On 401** — entire rebalance stops immediately; user is told which closes / opens succeeded; no "done" summary on a partial run.

Rebalance-specific:

- [ ] Diff is computed against current state (in dollars or weights, matching the user's target unit).
- [ ] Target plan validated like a fresh bulk plan: not mixed-unit, sum ≤ `CASH_ANCHOR` (or ≤ 100%), ≤ 20 instruments, all `instrumentID`s resolved.
- [ ] **Insufficient-cash variant: `close_target = shortfall + 1% of shortfall (rounded up)`** — closes are over-shot by this small buffer to avoid landing short and triggering a corrective second round.
- [ ] **Closes round UP** (free at least `close_target`, never less) — mirror image of the ceilings-on-opens rule.
- [ ] Phase 1 (closes) completes BEFORE Phase 2 (opens) begins.
- [ ] **60-second PnL cache wait between phases** — non-negotiable.
- [ ] Live cash re-verified after closes (`liveCash ≥ Σ planned_opens`); the buffer should make this comfortable — if short anyway, investigate (silent close failure? pending orders?) before proceeding.
- [ ] Rate-limit budget tracked across BOTH phases; no assumption of a fresh 20/min after the cache wait.
- [ ] Final state reported in the user's unit (dollars on main accounts; percentages in agent-portfolio context); pending-market-open status surfaced.
- [ ] For auto-rebalance: plan logged before execution so a stuck batch can be diagnosed later.
