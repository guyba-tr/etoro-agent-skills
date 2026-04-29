# Conditional / Rule-Based Trading

Reference for the `etoro-trading-assistant` skill. Load this when the user defines triggered actions — *"buy AAPL if it falls below $180"*, *"sell MSFT if up 10%"*, *"reduce BTC to $5,000 if external sentiment < 0.3"*.

This pattern wraps the standard single-trade open/close primitives (in `single-trade-walkthrough.md`) with a **polling/evaluation loop** plus optional external data sources. The trade execution itself is unchanged from the single-trade flow — what's new is the rule-shape, evaluation cadence, cooldowns, and safety controls around it.

---

## Account context

This workflow applies to **both main eToro accounts and agent-portfolios**. Examples below use **dollar amounts** — the main-account default.

> **Agent-portfolio override:** if you reached this reference from the `etoro-agent-portfolios` skill, apply **Override A** from that skill — replace dollar amounts with **percentages of equity** in every user-facing message (rule confirmations, trigger notifications, failure reports). Internally the rule's `size` field can hold either a dollar amount or a percentage; convert via `amount_usd = pct × EQUITY_ANCHOR` at trigger time. The condition definitions, polling cadence, cooldowns, and safety controls don't change.

---

## When to use

- User defines explicit triggers ("if X then Y").
- User wants the agent to monitor and act autonomously on their behalf.
- Recurring strategies that don't fit a one-time bulk action (those go in `bulk-trading.md`).

---

## 1. Capture rules in a structured form

Free-text rules are brittle. Translate the user's request into a structured object before activating it:

| Field | Example | Notes |
|---|---|---|
| `id` | UUID | For toggling/cancelling later |
| `type` | `'open'`, `'close'`, `'partial_close'`, `'reduce_to'`, `'increase_to'` | What action to take |
| `instrument` | `{ symbol: 'AAPL', instrumentID: 1001 }` | Resolve `instrumentID` up front |
| `condition` | `{ kind: 'price', op: '<', value: 180 }` | See condition kinds below |
| `target` | `{ dollars: 5000 }` or `{ weight: 0.05 }` | For `reduce_to` / `increase_to`; pick the unit that matches the user's framing |
| `size` | `{ dollars: 250 }` or `{ weight: 0.025 }` | For `open`; size at trigger time |
| `cooldown_s` | `3600` | Min seconds between re-triggers |
| `expires_at` | `'2026-12-31T00:00:00Z'` | Optional auto-disable time |
| `max_triggers` | `5` | Optional cap |
| `enabled` | `true` | Allow user to pause without deleting |
| `last_triggered_at` | `null` | Updated on each trigger |
| `trigger_count` | `0` | Updated on each trigger |

The unit on `size` and `target` (dollars vs weight) follows the user's framing: dollars on main accounts; weights in agent-portfolio context. Never mix the two within a single rule.

Always **echo the structured form back to the user for confirmation** before activating:

> *"I'll open a $250 AAPL position if the price drops below $180. I'll re-evaluate every 30s. After triggering once, I'll wait 1 hour before re-triggering, and I'll stop after 5 total triggers. Tell me 'pause AAPL rule' anytime to disable. Sound right?"*

---

## 2. Condition kinds and how to evaluate them

### `price` — live-price thresholds (most common)

Use the **rates endpoint**, NOT the PnL endpoint.

```
GET /market-data/instruments/rates?instrumentIds=<id>
```

Response: `{ rates: [{ instrumentID, ask, bid, lastExecution }] }`. For comparisons:

- For BUY conditions, compare against `ask` (you'll buy at the ask).
- For SELL conditions, compare against `bid` (you'll sell at the bid).
- For neutral references, the bid/ask midpoint is fine.

```typescript
const { rates } = await getRates([rule.instrument.instrumentID]);
const price = rate.lastExecution; // or ask/bid by direction
const conditionMet =
  (rule.condition.op === '<' && price < rule.condition.value) ||
  (rule.condition.op === '>' && price > rule.condition.value);
```

### `pnl_pct` — per-position P/L percentage

```
GET /trading/info/{env}/pnl
```

For the position:

```
pnl_pct = position.unrealizedPnL.pnL / position.amount × 100
```

Caveats from `account-snapshot.md`:

- For non-USD instruments, `position.amount` is the USD margin — `pnl_pct` is correctly USD-denominated, no native-currency adjustment needed.
- For leveraged positions, `position.amount` is the margin (cash committed), so `pnl_pct` is return-on-margin, not return-on-notional. Be explicit with the user about which one their rule means.
- The PnL endpoint's 60s cache means PnL-based triggers fire on data that's up to 60 seconds old. Acceptable for slow strategies; use `price` rules with `lastExecution` for fast-moving thresholds.

### `external` — third-party indicators

The agent calls the external API (sentiment, news score, custom signal), gets a number, compares.

- The agent owns the external integration; eToro just receives the resulting trade.
- **Cache external API responses** to avoid being rate-limited by the third party — sentiment scores rarely change second-to-second.
- Surface external-API failures to the user clearly: *"Sentiment API was unreachable; pausing rule until it recovers."*

---

## 3. Polling cadence

Each rule has its own cadence; combine cadences across rules to plan global polling:

| Trigger kind | Recommended cadence | Reason |
|---|---|---|
| `price` (live) | 30–60s | `/market-data/instruments/rates` may be cached server-side; faster polling doesn't yield fresher data and burns rate-limit budget. |
| `pnl_pct` | 60s | The PnL endpoint's own cache TTL. |
| `external` | Per third-party docs | Sentiment APIs typically recommend 5–15 min. |

The trade-execution rate limit (20 req/min) is the hard ceiling on **action**, not on monitoring. Monitoring uses the regular Public API which has its own (more generous) limits.

---

## 4. Trigger evaluation and execution

```typescript
async function evaluateRule(rule: Rule, ctx: Ctx) {
  if (!rule.enabled) return;
  if (rule.expires_at && Date.now() >= Date.parse(rule.expires_at)) {
    rule.enabled = false;
    notifyUser(`Rule ${rule.id} expired and was disabled.`);
    return;
  }
  if (rule.max_triggers && rule.trigger_count >= rule.max_triggers) {
    rule.enabled = false;
    notifyUser(`Rule ${rule.id} hit max-trigger cap (${rule.max_triggers}) and was disabled.`);
    return;
  }
  if (
    rule.last_triggered_at &&
    Date.now() - rule.last_triggered_at < rule.cooldown_s * 1000
  ) {
    return; // still in cooldown
  }

  const conditionMet = await evaluateCondition(rule.condition, ctx);
  if (!conditionMet) return;

  await dispatchAction(rule, ctx);  // calls openPosition / closePosition
  rule.last_triggered_at = Date.now();
  rule.trigger_count += 1;
  notifyUserOfTrigger(rule);
}
```

The `dispatchAction` call goes through `single-trade-walkthrough.md`'s flow — so all the standard rules apply:

- Resolve `instrumentID` (already done at rule creation; refresh if stale).
- `Leverage: 1` unless the user asked otherwise.
- Pre-flight available cash for `open` and `increase_to` actions.
- Use `UnitsToDeduct` correctly for `partial_close` and `reduce_to`.

**Each trigger fire IS its own workflow** — apply the full set of `execution-invariants.md` rules: read `/pnl` fresh, freeze `EQUITY_ANCHOR` + `CASH_ANCHOR` (§1), apply ceilings on size (§2), at-most-once on the POST (§3), stop on 401 (§4 — see "If the failure was a 401" in §6 below for the conditional-rules-specific handling). **Do NOT carry an anchor over from one trigger to the next** — between fires, the market drifts and so does equity.

For percentage-form rules (`size: { weight: 0.025 }`, common on agent-portfolios), `amount_usd = floor(weight × EQUITY_ANCHOR × 100) / 100`.

---

## 5. Safety controls

### Cooldowns (mandatory)

- Minimum default: **1 hour** per rule.
- Prevents rapid-fire triggering when a price oscillates around the threshold.
- For tight-band strategies, the user can request shorter cooldowns — but warn them about the noise risk.

### Per-rule trigger cap

- Recommend `max_triggers` ≤ 5 for any rule unless the user explicitly wants unbounded.
- Combined with cooldowns, this caps a runaway rule at "5 trades over 5 hours" for a 1h cooldown.
- This is the primary safety control. Per-rule caps + cooldowns are the runtime-friendly way to prevent runaway behavior; global caps and per-rule outcome-history circuit breakers would require persistent agent-side state that most runtimes don't expose to skills.

### Confirmation on activation

- Echo the structured rule back before turning it on (see §1).
- Tell the user **how to disable**: *"Reply 'pause AAPL rule' anytime."*
- Tell them how triggers will be communicated (see §6 below).

---

## 6. Communication when rules fire

Always immediate, in the user's unit. Main-account / dollar version:

```
Rule triggered: AAPL fell below $180 → opened $250 position.
- Status: filled at $179.50 (1.39 units).
- Triggers used: 1 of 5.
- Cooldown: 1 hour before this rule can re-fire.
```

If the trade is **pending market open** (markets closed; see `bulk-trading.md` §5):

```
Rule triggered: AAPL → $250 open.
- Status: pending market open (markets closed; will fill at open).
- Triggers used: 1 of 5.
```

If the trade **failed**:

```
Rule triggered: AAPL → tried to open $250.
- Status: FAILED — insufficient cash ($120 available; $250 needed).
- Rule remains enabled; consider freeing cash or reducing the size on this rule.
```

If the failure was a **401 Unauthorized** (the user's `x-user-key` was revoked):

- **Auto-disable every rule that uses the same credential.** They will all fail with the same error until the user provides a new key — leaving them enabled would generate a stream of 401s every polling cycle.
- Tell the user clearly: *"Rule X triggered AAPL → 2.5%, but the trade failed with 401 Unauthorized — your eToro API key appears to have been revoked. I've paused all your conditional rules until you share a new key."*
- Don't claim the trade happened. See `sso-and-session.md` §§3–4.

Agent-portfolio context substitutes percentages for the dollar amounts:

```
Rule triggered: AAPL fell below $180 → opened 2.5% allocation.
- Status: filled at $179.50.
- Triggers used: 1 of 5.
- Cooldown: 1 hour before this rule can re-fire.
```

---

## 7. Sanity checks

Cross-cutting invariants (covered by `execution-invariants.md`):

- [ ] **Each `dispatchAction` is its own workflow** — fresh `/pnl` read; `EQUITY_ANCHOR` + `CASH_ANCHOR` frozen for that single dispatch (§1); never carried over from a previous trigger or from rule creation.
- [ ] **For percentage-form rules** — stated weight is a CEILING (§2): `amount_usd = floor(weight × EQUITY_ANCHOR × 100) / 100`; over-fills surfaced and corrected.
- [ ] **At-most-once on the trigger POST** (§3) — only 429 retried; ambiguous outcomes reconciled at verification.
- [ ] **On 401** (§4) — the trigger trade is reported honestly; per §6, all rules using the same credential are auto-paused.

Conditional-rules-specific:

- [ ] Rules are stored in a structured form (not free-text), with `instrumentID` resolved at creation time.
- [ ] User confirms the rule shape before activation; "how to pause" is communicated.
- [ ] Live-price triggers use `/market-data/instruments/rates`; PnL-based triggers use the PnL endpoint at its 60s cadence.
- [ ] Cooldown set on every rule (default 1h) to prevent rapid re-firing.
- [ ] `max_triggers` defaulted to 5 unless user requested otherwise.
- [ ] Pending-market-open status surfaced when the triggered trade can't fill immediately.
- [ ] Communication uses the user's unit (dollars on main accounts; percentages in agent-portfolio context); units consistent throughout the rule's lifecycle.
