# eToro Account Snapshot

Reference for the `etoro-trading-assistant` skill. Apply whenever you read, compute, or display any data about the user's eToro account — balance, equity, open positions, copy-trading mirrors, pending orders, or P&L.

All of this data comes from a single endpoint at URL `/trading/info/{env}/pnl`, but the response object is `clientPortfolio` and contains the entire account snapshot — **not just a P&L figure**. Don't let the URL mislead you: anything you display about a user's account state passes through the formulas and rules below.

This reference covers two layers:

- **§1 — Account-level aggregation formulas** that match eToro's official calculation guides. Use them verbatim, or your numbers will disagree with the eToro UI.
- **§§2–5 — Per-field correctness rules** for values whose meaning isn't obvious from the field name alone.

The endpoint returns `{ clientPortfolio }` containing `credit`, `unrealizedPnL`, `positions[]`, `mirrors[]`, `orders[]`, `ordersForOpen[]`.

## 1. Account-level aggregation formulas

### Available Cash

```
Available Cash = credit
               − Σ(ordersForOpen[i].amount where mirrorID === 0)
               − Σ(orders[i].amount)
```

Filter `ordersForOpen` to manual positions only (`mirrorID === 0`); the `orders` array is summed in full regardless of `mirrorID`.

### Total Invested

```
Total Invested = Σ(positions[i].amount)
               + Σ(mirrors[i].positions[j].amount)
               + Σ(mirrors[i].availableAmount − mirrors[i].closedPositionsNetProfit)
               + Σ(ordersForOpen[i].amount where mirrorID === 0)
               + Σ(orders[i].amount)
               + Σ(ordersForOpen[i].totalExternalCosts where mirrorID === 0)
```

The `mirrors[i].availableAmount − closedPositionsNetProfit` term subtracts realized profit from closed copy positions (which is otherwise double-counted via the Profit/Loss formula below). The `totalExternalCosts` term covers fees on manual pending orders.

### Profit / Loss (Unrealized PnL)

```
Profit/Loss = Σ(positions[i].unrealizedPnL.pnL)
            + Σ(mirrors[i].positions[j].unrealizedPnL.pnL)
            + Σ(mirrors[i].closedPositionsNetProfit)
```

The third term is the realized profit from closed copy positions — that's why Total Invested above subtracts it from `availableAmount` (so it isn't counted twice in Equity).

### Equity

```
Equity = Available Cash + Total Invested + Profit/Loss
```

All three components are USD-only.

Source guides (authoritative — refer to these if a formula needs to evolve):

- <https://api-portal.etoro.com/guides/calculate-available-cash>
- <https://api-portal.etoro.com/guides/calculate-total-invested>
- <https://api-portal.etoro.com/guides/calculate-profit-loss>
- <https://api-portal.etoro.com/guides/calculate-equity>

### Stability of these values between snapshots

Assuming **single-actor mode** (no other client opens or closes positions on this account):

- **Available Cash is STABLE** between trades — it changes only when the agent itself opens, closes, or modifies a position (or when an `ordersForOpen` order fills). If you read it now and don't trade, it'll be the same value next time you read it.
- **Total Invested is STABLE** by the same logic — it's the sum of position cost-bases, which only change on open/close.
- **Profit/Loss DRIFTS continuously** with market price movements on open positions. Even with no trading activity, this number changes second-to-second.
- **Equity DRIFTS** because it includes Profit/Loss. Same reason.

This matters for any workflow that expresses intent as a percentage (*"30% in BTC"*). Because equity drifts, recomputing `pct × current_equity` mid-workflow produces a different dollar amount than it did at the start — making the same intent resolve to different numbers depending on timing. **For workflows requiring percentage stability, freeze `EQUITY_ANCHOR` (and `CASH_ANCHOR`) at workflow start and use them for all sizing, sufficiency, and verification.** Full statement of the anchor-freeze invariant: `execution-invariants.md` §1. Workflow-specific application: `bulk-trading.md` §2, `rebalancing.md` §1, `conditional-rules.md` §4. Agent-portfolio strict-application override: `etoro-agent-portfolios` SKILL "Override B".

### 60-second response cache (rolling)

Independent of the underlying data's stability, the `/pnl` endpoint **caches the entire response for 60 seconds (rolling)**. This is the most consequential operational fact about the endpoint and it's easy to miss, because the URL is named for a single field while the response carries the whole `clientPortfolio`.

**The cache covers every field in the response, not just the derived aggregates:**

- `positions[]` — including newly opened or newly closed positions
- `ordersForOpen[]` — including newly placed pending orders
- `mirrors[]`
- `credit`, `unrealizedPnL`
- and therefore everything derived from them (Available Cash, Total Invested, Profit/Loss, Equity from §1 above)

**Practical consequence:** if the agent executes a trade and then reads `/pnl` seconds later, the response is the *pre-trade* snapshot — the just-opened position will NOT appear in `positions[]`, and the just-closed position will still be there with its old `units`. This is **not** a bug to work around with retries; it is the documented cache behavior and there is no `force-refresh` or `cache-bust` parameter.

**The rule:** for any post-trade verification read (categorizing fills vs. pending vs. failed; reconciling ambiguous outcomes from `execution-invariants.md` §3; checking that closes freed enough cash for a follow-on open; computing post-trade Available Cash for the next workflow's anchor), **wait 60 seconds after the last write before reading `/pnl`**. Reading earlier returns stale data and produces false "trade didn't land" reports.

This is what justifies the explicit waits in the workflow references: `single-trade-walkthrough.md` Step 6, `bulk-trading.md` §5, `rebalancing.md` §6 + §8.

**For monitoring (not verification),** the same cache means PnL-derived triggers (`conditional-rules.md` §2 `pnl_pct`) operate on data up to 60 s old — acceptable for slow strategies; use `/market-data/instruments/rates` (a different endpoint, not subject to this cache) for fast price-threshold triggers.

## 2. `position.openRate` is in the instrument's NATIVE currency

For **non-USD** instruments (e.g. `BP.L` quoted in pence, SEK / NOK / HKD / etc.), `openRate` is in that instrument's native currency, **not USD**. To get the USD-equivalent price for cash positions:

```
usdPriceEquivalent = position.amount / position.units
```

(The wire `amount` is in USD; `units` is in shares/contracts.)

Always normalize before displaying or sorting on price — otherwise BP.L will appear ~125× cheaper than AAPL because pence aren't pounds and aren't dollars.

## 3. For LEVERAGED positions, `amount / units` is margin-per-unit, NOT market price

eToro reports `amount` as the **margin** (cash committed) and `units` as the size of the underlying exposure. For unleveraged positions (`leverage === 1`) margin happens to equal notional, so `amount / units` coincidentally equals the entry price — but for leveraged positions the notional is `amount × leverage`, and dividing by `units` understates the true price by exactly that factor (e.g. 5× leverage gives a "price" 5× too low).

**Rules:**

- Detect leveraged positions via `position.leverage > 1`.
- Use `position.openRate` directly as the entry price for leveraged positions.
- **Don't synthesize a current/live rate from PnL.** Formulas like `openRate ± pnl/units` drift from eToro's own valuation and will disagree with the eToro UI. If you need a current market rate, read it from a price endpoint (`/market-data/instruments/rates`) — never reconstruct it from PnL fields.
- **General principle:** display only fields the endpoint returns explicitly. Don't synthesize numerics the API didn't give you.

## 4. The "fees" figure blends actual fees with dividends — and they are NOT separable

`totalFees` (and the related `totalExternalFees` / `totalExternalTaxes`) is internally `actual_fees − dividends_received`. The API does not expose the two components individually. Consequences differ by instrument type:

- **Crypto, Commodities, Currencies, Indices** — these don't pay dividends, so the field is always non-negative and reads cleanly as "fees." Safe to label as such.
- **Stocks and ETFs** — the field can be positive, zero, or negative depending on whether accumulated fees exceed accumulated dividends. A negative value means dividends have outpaced fees so far; a positive value can still hide a significant dividend offset (you can't tell how much).

**Rules:**

- **Don't clamp negative values to zero.** That silently discards real dividend income.
- **Aggregating across mixed instrument types is misleading.** A small group total may reflect either low fees or large dividend credits cancelling them out, and the user can't distinguish.
- **When surfacing this data for stocks or ETFs (or any group that contains them), disclose the blend to the user** — e.g. label the column "Fees − Dividends (net)" or attach a tooltip. For dividend-incapable instrument types you can label the column "Fees" with no caveat.

## 5. Copy (mirror) positions appear in TWO arrays — pick ONE

The same copy positions live in:

- **`clientPortfolio.positions[]`** — flat list, identifiable by `mirrorID > 0`.
- **`clientPortfolio.mirrors[].positions[]`** — grouped-by-trader re-projection of the same data.

Pick one source of truth for display and aggregation, or you will double-count. Recommendation: use `clientPortfolio.positions[]` as the authoritative source and treat `mirrors[]` as a UI grouping helper only (e.g. for showing a "by trader" view).

When **collecting instrument IDs for enrichment** (e.g. fetching display names and logos), walk both arrays — that way no copy-position instrument is missed even if your transformation later picks one source for display.

> The aggregation formulas in §1 already handle this correctly — they sum `mirrors[i].positions[j]` separately from top-level `positions[]`, so following them verbatim avoids the double-count by construction.

## Quick checklist before showing any account-state info to the user

- [ ] Account-summary numbers (Available Cash / Total Invested / P&L / Equity) match the §1 formulas exactly — no shortcuts that drop the `closedPositionsNetProfit` adjustment or `totalExternalCosts` term.
- [ ] Prices for non-USD instruments are converted via `amount / units`, not displayed as raw `openRate`.
- [ ] Leveraged positions use `openRate` as the entry price, not `amount / units`.
- [ ] No synthesized "current price" computed from PnL — only fields the API returned explicitly.
- [ ] Fees columns containing stocks or ETFs are labelled to reflect the dividend blend; no clamping of negative values to zero.
- [ ] When you display only one source of positions, that source is `positions[]`; `mirrors[].positions[]` is not summed alongside it.
