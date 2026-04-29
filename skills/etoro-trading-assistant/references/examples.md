# Worked Examples

Reference for the `etoro-trading-assistant` skill. Four end-to-end conversation walkthroughs that show the rules from `execution-invariants.md` and the workflow references applied in practice — the tone, the pacing of confirmations, the order of steps, and what the user actually sees.

These are illustrative, not normative. Where they conflict with a workflow reference, the reference wins.

---

## Example 1 — Single-trade open on a main account

**Scenario:** mid-conversation, the user (already authenticated with an API user-key for their main eToro account) decides to buy AAPL. The environment (real vs demo) is determined from the key — the agent does not ask.

```
USER: I want to buy $1,000 of AAPL.
```

**Agent's internal steps** (not shown to the user):

1. *Capture intent* (`single-trade-walkthrough.md` Step 1): action=open, symbol=AAPL, size=$1,000, direction=buy. All four required fields present.

2. *Resolve instrument ID* (Step 2). Not in cache.

   Internal call: `GET /market-data/search?internalSymbolFull=AAPL` → `{ items: [{ instrumentId: 1001, internalSymbolFull: "AAPL", ... }] }`. Exact match on `AAPL`. Cache it.

3. *Pre-flight cash* (Step 3). Read `/pnl` for the active environment (per `api-conventions.md` "Demo vs. real environments" — the env was already determined the first time the agent hit `/pnl` this session). Compute Available Cash via `account-snapshot.md` §1 → $5,237. $1,000 ≤ $5,237 → proceed.

4. *Confirm with the user* (Step 4) — flag the ~1-minute verification wait up front:

```
ASSISTANT: Ready to place: buying $1,000 of AAPL at market, leverage 1× (no leverage).
I'll place the order and confirm the result in about a minute.
Proceed?

USER: yes
```

5. *Execute* (Step 5):

   - Endpoint: `POST /trading/execution/{env}/market-open-orders/by-amount` (with `{env}` = the determined environment, e.g. `real`).
   - Body: `{ "InstrumentID": 1001, "IsBuy": true, "Leverage": 1, "Amount": 1000 }`
   - Headers: `x-api-key`, `x-user-key`, `x-request-id` (fresh UUID v4).
   - Response: `200 { "orderId": "..." }` → at-most-once: done, move on (`execution-invariants.md` §3).

6. *Wait then verify* (Step 6).

   - `sleep(60_000)` — the `/pnl` endpoint is cached for 60 s and the cache covers `positions[]` / `ordersForOpen[]`; reading earlier returns the pre-trade snapshot (see `account-snapshot.md` §1 "60-second response cache").
   - Re-read `/pnl`. New AAPL entry in `clientPortfolio.positions[]` at $1,000 / 5.57 units / `openRate` $179.50. Filled.

7. *Report back* (Step 7) — single bottom-line message after the wait, in the user's framing (dollars + units), with cash-impact side effect:

```
ASSISTANT: Bought $1,000 of AAPL at $179.50 (5.57 units). Available cash dropped from $5,237 to $4,237.
```

The user sees one short message during execution (Step 4 confirmation, with the "about a minute" expectation) and one bottom-line message after verification (Step 7). No interim "Order placed; verifying…" — just stay quiet and deliver the verified result.

### What you do NOT see in this example

- **No "real or demo?" question.** The user-key is bound to one environment at creation; the agent already knows. (`api-conventions.md` "Demo vs. real environments".)
- **No anchor freeze ceremony.** For a single dollar-form trade with explicit cash pre-flight, the workflow doesn't need a frozen `EQUITY_ANCHOR` (the dollar amount IS the user's intent, not a derived one). If the user had said *"buy 5% of equity in AAPL"*, the agent would freeze `EQUITY_ANCHOR` at the start of Step 3, compute `amount_usd = floor(0.05 × EQUITY_ANCHOR × 100) / 100`, and verify the fill against that anchor (`execution-invariants.md` §§1–2).
- **No second-guessing on a 200 response.** The agent does not re-fire the same payload "to be safe" — that's the at-most-once rule.
- **No early `/pnl` read.** Verifying before the cache refreshes would show the *pre-trade* snapshot (no AAPL in `positions[]`) and produce a false "trade didn't land" report. The 60 s wait is what makes the verified report trustworthy.
- **No two-message pattern.** The agent does not say *"Order placed; verifying…"* and then follow up. The user gets the Step 4 expectation-set ("about a minute") and the Step 7 verified result — nothing in between.

---

## Example 2 — Bulk build on an agent-portfolio (8 positions)

**Scenario:** the user has just created an agent-portfolio (per `etoro-agent-portfolios` skill, `references/onboarding.md`). Both the `etoro-trading-assistant` and `etoro-agent-portfolios` skills are loaded. Override A (percentages, never dollars) and Override B (always read live `/pnl` first) apply throughout.

```
USER: Build me a portfolio: 25% AAPL, 15% MSFT, 10% GOOGL, 20% BTC, 10% ETH, 10% NVDA, 5% SOL. Keep 5% in cash.
```

### Phase 0 — anchor freeze (Override B + `execution-invariants.md` §1)

Internal: `GET /trading/info/real/pnl` (with the agent-portfolio's `userToken` as `x-user-key`).

```
EQUITY_ANCHOR = $9,847.23   // computed via account-snapshot.md §1
CASH_ANCHOR   = $9,847.23   // empty portfolio, all cash
```

These anchors are now frozen for the entire build. Note the user-facing message (next) does NOT mention either of these dollar values — Override A.

### Phase 1 — validate the plan (`bulk-trading.md` §1)

- Single unit (percentages) — OK.
- `Σ percentages = 25 + 15 + 10 + 20 + 10 + 10 + 5 = 95` ≤ 100% — OK. Remaining 5% is the user's stated cash buffer.
- 7 instruments — well under the recommended 20.
- Resolve all 7 symbols → `instrumentID`s up front. None fail. Cache them.

### Phase 2 — sizing (`execution-invariants.md` §2 ceilings)

For each instrument, `amount_usd = floor(pct × EQUITY_ANCHOR × 100) / 100`:

| Symbol | pct | amount_usd |
|---|---|---|
| AAPL | 25% | $2,461.80 |
| MSFT | 15% | $1,477.08 |
| GOOGL | 10% | $984.72 |
| BTC | 20% | $1,969.44 |
| ETH | 10% | $984.72 |
| NVDA | 10% | $984.72 |
| SOL | 5% | $492.36 |

`Σ amount_usd = $9,354.84` ≤ `CASH_ANCHOR` ($9,847.23) → sufficiency check passes (`bulk-trading.md` §2).

**Open-buffer check** (`execution-invariants.md` §2 "Open buffer"): `planned_cash_pct = ($9,847.23 − $9,354.84) / $9,847.23 = 5.0%`. That's well above 1%, so **no buffer is applied** — the user's stated 5% cash reserve already absorbs eToro's per-trade fees naturally. (If the user had asked for a 99.5% allocation leaving 0.5% cash, the buffer would have triggered and each `amount_usd` above would have been re-floored to `× 0.99`, silently — percentage-form bulk plans don't disclose the under-fill since it's invisible at percentage resolution.)

### Phase 3 — confirm with the user (Override A — percentages, not dollars)

```
ASSISTANT: I'll open 7 positions on your agent-portfolio:
  - AAPL:   25%
  - MSFT:   15%
  - GOOGL:  10%
  - BTC:    20%
  - ETH:    10%
  - NVDA:   10%
  - SOL:     5%
  - (cash):  5%

Total allocated: 95%. Estimated time ~85 seconds (~25 s of 3s-paced execution + 60 s wait for the PnL cache to refresh before verification).

Proceed?

USER: go
```

### Phase 4 — execute, paced ≥ 3s (`bulk-trading.md` §4)

Per-trade outcomes captured as `'ok' | 'failed' | 'ambiguous' | 'rate_limited_giveup'` (`execution-invariants.md` §3).

| Trade | Outcome |
|---|---|
| AAPL ($2,461.80) | 200, orderId — `ok` |
| MSFT ($1,477.08) | 200, orderId — `ok` |
| GOOGL ($984.72) | 200, orderId — `ok` |
| BTC ($1,969.44) | 200, orderId — `ok` |
| ETH ($984.72) | 200, orderId — `ok` |
| NVDA ($984.72) | timeout, no response — `ambiguous` (do NOT retry) |
| SOL ($492.36) | 200, orderId — `ok` |

### Phase 5 — wait 60s, verify (`bulk-trading.md` §5)

Internal: sleep 60s → `GET /trading/info/real/pnl`.

For each trade, look in `positions[]` (filled) or `ordersForOpen[]` (pending market open):

- AAPL, MSFT, GOOGL, BTC, ETH, SOL → all in `positions[]`. Filled.
- NVDA → also found in `positions[]` at $984.72. The order DID land despite the ambiguous response. Reclassify as `ok`. (This is the at-most-once verification reconciliation in action.)

Over-allocation check (`execution-invariants.md` §2 verify-against-anchor): for each filled position, `actual_amount == expected_amount` exactly. No over-fills.

### Phase 6 — report back (Override A — percentages)

```
ASSISTANT: Build complete on your agent-portfolio:
  - Filled: 7 positions (AAPL 25%; MSFT 15%; GOOGL 10%; BTC 20%; ETH 10%; NVDA 10%; SOL 5%).
  - Pending market open: 0.
  - Failed: 0.

Your portfolio is 95% allocated; the remaining 5% is in cash as you specified.

Note: NVDA's request had a network hiccup (no immediate response) but the order did land — verified at the post-build cache refresh.
```

### What this example illustrates

- **Override A everywhere user-facing.** Internally, `Amount: 2461.80` was sent in the API call; the user only ever saw "25%".
- **At-most-once on the NVDA timeout.** No retry was fired; the order was reconciled by reading `positions[]`. Had the order NOT landed, the agent would have surfaced *"NVDA didn't land — would you like me to retry it?"* and waited for explicit consent.
- **Anchor-locked sizing.** Even if BTC's price moved 2% during the 25-second build, the `BTC: 20%` line resolves to `$1,969.44` in the API call and on the verification check — because both reference `EQUITY_ANCHOR`, not current equity.
- **No "$9,354.84 deployed" in the user-facing report.** That number exists in the agent's working memory as `Σ amount_usd`; the user-facing equivalent is "95% allocated."

---

## Example 3 — 401 mid-bulk-build (the don't-hallucinate rule in action)

**Scenario:** same setup as Example 2, but at trade 4 of 7, the user's `userToken` returns 401 (they revoked it in another tab while the build was running).

After AAPL, MSFT, GOOGL succeeded:

| Trade | Outcome |
|---|---|
| AAPL ($2,461.80) | `ok` |
| MSFT ($1,477.08) | `ok` |
| GOOGL ($984.72) | `ok` |
| BTC ($1,969.44) | **401 Unauthorized** |

`execution-invariants.md` §4 + `sso-and-session.md` §3.a: stop the entire batch immediately. Do NOT continue with ETH, NVDA, SOL — they would all fail the same way.

```
ASSISTANT: Your eToro API key for this agent-portfolio appears to have been revoked (the API returned 401 Unauthorized).

What I executed BEFORE the failure:
  - AAPL:  25% — opened
  - MSFT:  15% — opened
  - GOOGL: 10% — opened

What I did NOT execute (no order was placed):
  - BTC:  20%
  - ETH:  10%
  - NVDA: 10%
  - SOL:   5%

There's no refresh for an agent-portfolio user-token. If the agent-portfolio still exists and you want to keep this allocation, you'll need to share a fresh user-token (or, if the original token is unrecoverable, create a new agent-portfolio per the onboarding flow). Once we have a working token I can either resume from BTC onward or revise the plan first — your call.
```

What this would NOT look like:

- ❌ *"Build complete"* (3 of 7 is not complete).
- ❌ *"Some trades may have failed, please check your account"* (vague; the agent knows exactly which ones).
- ❌ *"Retrying…"* (no point — the credential is dead, retrying just generates more 401s).
- ❌ Auto-resuming after the user provides a new token without explicit consent (the market may have moved; the user may want to revise).

---

## Example 4 — Close all positions (talking in bottom lines, not mechanics)

**Scenario:** the user asks the agent to close all 10 open positions on their agent-portfolio. The agent's recovery work mid-execution — endpoint-shape discovery, payload fixes, partial-batch recovery — should be **completely invisible** to the user. The user gets one confirmation, then silence, then the verified result.

This example exists because the failure mode it prevents is the most common UX bug in trading agents: the agent narrating its own internal mechanics, retries, and bug-fixes to the user as if the user were a developer debugging alongside it.

```
USER: Close all my open positions.
```

### ❌ Bad version — what the agent should NOT send

These are the kinds of messages that leak internal mechanics to the user. Every one of them is technically true but operationally noise:

```
ASSISTANT: Now let me close all 10 positions. Agent-portfolio = always
real environment, at-most-once on each POST, and I need to pace them
(rate limits: ≥3s between requests).

ASSISTANT: Found it. The real close endpoint doesn't use /real/ in the
path. It's just /trading/execution/market-close-orders/positions/{positionId}.

ASSISTANT: The close endpoint requires InstrumentId in the body. Let me
fix that and run all 10.

ASSISTANT: First one (BTC) closed successfully. The loop broke early.
Let me close the remaining 9.

ASSISTANT: All 10 positions closed successfully. Let me verify.

ASSISTANT: All clear. 10/10 positions closed, all filled. Portfolio is
now 100% cash.
```

Why each line is wrong:

- **Line 1** narrates invariants (real environment, at-most-once, pacing). The user does not need to know about HTTP rate limits to read a status update.
- **Line 2** is the agent debugging an endpoint path. The user neither knows nor cares what `/real/` is.
- **Line 3** is the agent fixing its own request payload bug. Internal.
- **Line 4** exposes loop control flow — *"the loop broke early"* tells the user something went wrong without telling them what to do about it. If nothing was actually lost, don't mention it.
- **Line 5** is a status ping that adds nothing for the user.
- **Line 6** is the **only** line the user actually needed.

### ✅ Good version — what the agent SHOULD send

```
ASSISTANT: I'll close all 10 of your open positions. Estimated time
~90 seconds (~30s of paced execution + 60s wait for the PnL cache to
refresh before verification). Proceed?

USER: yes

[~90 seconds of silence — agent executes, recovers from any payload
 or path issues silently, retries 429s silently, waits 60s, verifies]

ASSISTANT: Done — all 10 positions closed. Your portfolio is now 100%
cash.
```

Two messages, total. One sets expectations and asks for consent. One delivers the verified bottom line.

### What changed between the two versions

The agent does the **same internal work** in both cases:
- Resolves the close endpoint shape and required fields
- Paces ≥ 3s between POSTs (per `bulk-trading.md` §4)
- Recovers silently if a request returns 4xx with a fixable payload error (within at-most-once: a payload-shape fix is a *different* request, not a retry of the same one — so it's allowed)
- Waits 60s for the cache (per `account-snapshot.md` §1 "60-second response cache")
- Reads `/pnl` and confirms 0 open positions remaining

What changes is **what the agent says out loud about that work**. In the good version: nothing. In the bad version: all of it.

### When silence is wrong — escape hatches

The "stay silent" rule has three legitimate exceptions, all called out in `etoro-trading-assistant/SKILL.md` "Talk in bottom lines, not mechanics":

1. **Destructive choice that requires user input.** *"You have two AAPL positions, opened at different times. Close both, or just one?"* — wait for the user. Don't pick.
2. **Credential failure mid-batch (401).** Stop immediately and report which closes succeeded and which never executed (per `execution-invariants.md` §4 and Example 3 above). Never claim a partial close batch is "done".
3. **Partial outcome the user needs to know about.** If 9 of 10 closed and NVDA failed for a non-recoverable reason, the bottom line includes the failure: *"Closed 9 of 10. NVDA failed (insufficient permissions on this instrument). Want me to skip it or escalate?"* — *not* *"Let me retry NVDA"* (which is internal).

In all other cases — payload fixes, endpoint-path discovery, 429 backoffs, ambiguous-outcome reconciliation — the user sees nothing until the verified bottom line.
