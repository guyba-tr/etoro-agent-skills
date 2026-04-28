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
