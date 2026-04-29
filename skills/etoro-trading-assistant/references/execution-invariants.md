# Execution Invariants

Reference for the `etoro-trading-assistant` skill. Four cross-cutting rules apply to **every** trade-execution workflow in this skill — single trade, bulk build, rebalance, conditional-rule trigger — regardless of whether you're operating on a main account or an agent-portfolio.

Each workflow reference points back here for the canonical statement; workflow-specific applications (e.g. the close-buffer in `rebalancing.md`, the cumulative `spent_so_far` check in `bulk-trading.md`) are documented in their own files.

---

## 1. Anchor freeze — equity and cash are read once per workflow

Before sizing or executing anything in a workflow, fetch the account state and **freeze two anchors** that don't change for the workflow's duration:

```
pnl = GET /trading/info/{env}/pnl

EQUITY_ANCHOR = equity(pnl)         // unit of user intent: "X% in symbol Y" means X% × EQUITY_ANCHOR
CASH_ANCHOR   = available_cash(pnl) // execution constraint at workflow start
```

Compute both via the formulas in `account-snapshot.md` §1.

**Rules:**

- Use these anchors for ALL sizing, sufficiency, and verification within the workflow — never recompute from current equity mid-flow.
- A new workflow gets a **fresh** `/pnl` read and its **own** anchors. Don't carry anchors over from one workflow to the next, or from rule creation to rule trigger.
- On multi-phase workflows (e.g. rebalancing's close-then-open), anchors stay frozen across phases. The 60-second PnL-cache wait between phases is what makes the post-close `/pnl` read return current state at all (the cache covers the entire `clientPortfolio`, including `positions[]` and `ordersForOpen[]` — see `account-snapshot.md` §1 "60-second response cache"). The wait re-reads cash for **execution-readiness** (a live check that closes settled), but does **not** re-anchor.

**Why freeze.** Equity drifts continuously with market movement on existing positions; cash is stable in single-actor mode (no other client opening or closing positions on the account). If the agent recomputed `pct × current_equity` mid-flow, the same intent ("30% in BTC") would resolve to different dollar amounts depending on when the calculation ran — and the post-fill percentage would slide too. Freezing makes percentages deterministic and verifiable. See `account-snapshot.md` §1 "Stability of these values between snapshots" for the underlying drift model and "60-second response cache" for the cache mechanic.

---

## 2. Stated allocations are CEILINGS, never targets

Every percentage and every dollar amount the user agreed to is an **upper bound**. The actual filled allocation must be ≤ the stated figure. **Always under-fill rather than over-fill** — a small under-allocation is invisible and recoverable; an over-allocation requires an unwinding partial close, surprises the user, and can incur fees on the corrective trade.

### Three rules together guarantee the ceiling on opens

1. **Floor when converting percentage → dollars.** Anchor on `EQUITY_ANCHOR` from §1, NOT current equity. Floor to cents — never round to a "nice number" like $300 when the math gives $295.40.
   ```
   amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100
   ```
2. **Cumulative check before each send** (multi-trade workflows). Maintain `spent_so_far`. Before firing trade `i`:
   ```
   spent_so_far + amount_usd[i] ≤ Σ planned amount_usd
   spent_so_far + amount_usd[i] ≤ CASH_ANCHOR
   ```
   If either constraint would be violated, that's an arithmetic bug in the planner — abort and recompute. Do **not** silently shrink-and-fire (the user agreed to a specific plan).
3. **Send the floored dollar amount.** Since eToro has no amount-slippage (`Amount: $X → position.amount = $X` exactly), the floored amount IS the exact filled value. The percentage of `EQUITY_ANCHOR` lands at or just under the stated cap.

For **dollar-form intents** the same discipline applies in mirror image — *"buy $295 of BTC"* means send `$295` exactly, never round up to `$300` for cleanliness.

### Verify against the anchor, not against current equity

After the order fills, for every position that landed in `positions[]`:

```
expected_amount = floor(stated_pct × EQUITY_ANCHOR × 100) / 100  // or the user's stated dollar figure
actual_amount   = position.amount

if actual_amount > expected_amount:
  // CEILING VIOLATION — agent-side bug. Surface it and offer a corrective partial close.
```

Equity may have moved between the anchor freeze and the verification read; verifying against current equity would falsely flag (or falsely pass) positions whose dollar size is exactly correct. The anchor is the user's intent; current equity is the moving market.

When over-fill is detected: present clearly and offer the fix.

```
⚠ BTC filled at $305 (30.5% of equity). Stated cap was 30% ($300).
This is an over-allocation by $5. Trim by partial-closing $5 of BTC to bring it to exactly 30%? [Y/n]
```

Compute `over = actual_amount − expected_amount` and partial-close `over` worth of the position. The corrective close must itself follow §3 (at-most-once); re-verify via `/pnl`.

### Mirror-image rule for closes when freeing cash

When the close is happening to **free cash for an upcoming open** (the insufficient-cash variant in `rebalancing.md` §A), the closes round **UP** instead of down — close at least `shortfall + buffer`, never less. Without that, partial-close unit rounding or post-close fees can leave you a few dollars short of the target and force a corrective second close-then-wait round. Full mechanics: `rebalancing.md` §A.

---

## 3. At-most-once delivery on every trade-execution POST

**Trade-execution endpoints have no idempotency key.** The eToro API offers no header or body field that lets you safely re-send the same trade. Resending after an ambiguous outcome can — and does — produce duplicate positions. Treat every trade-execution request with the **at-most-once** discipline below.

### Status-code → action table

| Outcome | Meaning | Action |
|---|---|---|
| **2xx** with an `orderId` (or equivalent) | Server accepted and recorded the order | **Done. Don't resend. Move on to verification.** |
| **4xx** (except 429) — 400, 403, 404, etc. | Server rejected explicitly; nothing was placed | Log as failed; surface to the user; **do not retry the same payload** (a 400 won't become a 200) |
| **401** | Credentials invalid | **STOP the workflow immediately.** See §4 below. |
| **429** | Rate-limited; server explicitly told you it didn't process | Safe to retry — see "429 backoff" below. The only safe retry case. |
| **5xx** with a body | Server processed and failed | Log as failed; **do not retry** — you don't know whether the order was placed before the failure |
| **Timeout / connection reset / no response / parse error** | Truly ambiguous — the order may or may not have been placed | **Do not retry.** Mark as ambiguous, continue the workflow, and reconcile via `/pnl` at verification |

### Why "do not retry" on ambiguous and 5xx outcomes

The request might have reached the server, executed, and the response was lost on the way back. A retry in that case opens a duplicate position — exactly the failure mode that surprises users. **A missing position is recoverable** (verify, then place again deliberately); **a duplicate is messy to unwind**.

The only response that justifies a same-payload retry is **429** (the server explicitly tells you it didn't process the request). Everything else either succeeded, failed cleanly, or is ambiguous — and ambiguous gets resolved at verification time, not by hopeful re-firing.

### 429 backoff cadence

- First 429: wait **15 seconds**, retry the same trade.
- Second 429 on the same trade: wait **30 seconds**, retry.
- Third: wait **60 seconds**, retry.
- After three failures, mark the trade as failed and continue with the next.

### Reconciling ambiguous trades at verification

After workflow execution completes, read `/pnl` and check whether each ambiguous trade appears in `positions[]` or `ordersForOpen[]`:

- **Found in either array** → the order *did* land despite the ambiguous response. Reclassify as filled or pending. **Do not** place another order — that's the duplicate-prevention point.
- **Not found in either array** → the order didn't land. Now (and only now) is it safe to ask the user whether to retry it as a fresh trade.

### Per-trade outcome status (multi-trade workflows)

Capture a four-state status, not just ok/failed:

```typescript
type TradeStatus =
  | 'ok'                   // explicit 2xx with orderId
  | 'failed'               // explicit 4xx/5xx response
  | 'ambiguous'            // no response, timeout, connection reset — actual outcome unknown
  | 'rate_limited_giveup'; // 429 retried 3× and still failed
```

---

## 4. On 401 — stop the workflow immediately; never hallucinate trades

A 401 from the Public API means the credentials weren't accepted. The recovery path depends on which auth mode the agent is using (see `sso-and-session.md` §3 for the full branching). **Regardless of auth mode, the workflow rule is identical: STOP.** Don't keep firing requests; they'll all fail with the same 401.

### Mid-workflow 401 — the don't-hallucinate rule

If a 401 hits mid-workflow (during a bulk build, rebalance, or sequence of conditional-rule triggers), the trades that fired BEFORE the 401 likely succeeded; the ones AFTER the 401 will all fail with the same error. The agent **must**:

- **Stop the workflow.** Do not continue executing — every subsequent trade will hit the same 401.
- **Tell the user exactly which trades executed and which didn't.** Be explicit, name the instruments and amounts, don't gloss over the partial state.
- **Do not summarize the workflow as if it completed.** Phrases like *"build complete"* or *"rebalance done"* when only some trades fired are the worst possible UX — the user will think they own positions they don't, or vice versa.
- **Do not retry the failed trades after the user fixes the key without explicit confirmation.** The market may have moved; the user may want to revise the plan.

Example, after a 401 interrupts a bulk build at trade 4 of 8:

```
Your eToro API key appears to have been revoked (the API returned 401 Unauthorized).

What I executed BEFORE the failure:
- AAPL: $2,500 — opened
- MSFT: $1,500 — opened
- GOOGL: $1,000 — opened

What I did NOT execute (no order was placed):
- BTC: $2,000
- ETH: $1,000
- NVDA: $800
- TSLA: $700

Please verify your API key at https://www.etoro.com/settings/trade and share a new one. Then we can decide whether to execute the remaining 4 positions or revise the plan.
```

For the full per-auth-mode recovery flow (no-refresh on API-key auth; single-refresh-attempt on SSO/Bearer), see `sso-and-session.md` §§3–4.

---

## Quick reference — invariants checklist

Every workflow reference includes its own sanity-checks at the bottom that re-state the workflow-specific application of these invariants. As a quick standalone checklist:

- [ ] **Anchor freeze**: `/pnl` read at workflow start; `EQUITY_ANCHOR` and `CASH_ANCHOR` frozen for the duration; never recomputed mid-flow.
- [ ] **Ceilings**: `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100`; cumulative `spent_so_far + next ≤ Σ planned ≤ CASH_ANCHOR` checked before each send; post-fill verification against `EQUITY_ANCHOR` (not current equity); over-fills surfaced and corrected via partial close.
- [ ] **At-most-once**: only 429 triggers a same-payload retry; 4xx (non-429), 5xx, and ambiguous outcomes are logged and reconciled by reading `positions[]`/`ordersForOpen[]` at verification time, never by re-firing.
- [ ] **401**: workflow stops immediately; the user is told which trades succeeded and which never executed; no "completed" summary on a partial run.
