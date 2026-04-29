# eToro API Conventions

Reference for the `etoro-trading-assistant` skill. These are foundational facts about how the eToro Public API works — they apply to virtually every request.

## Two distinct hosts

eToro splits its endpoints across two hosts with different content types and error envelopes. **You cannot share one client wrapper between them.**

| Concern | Public API | SSO / STS |
|---|---|---|
| Base URL | `https://public-api.etoro.com/api/v1` | `https://www.etoro.com/sso/...` (and adjacent OAuth endpoints) |
| Request body | `application/json` | `application/x-www-form-urlencoded` |
| Error envelope | Free-form `{ error: string }` plus HTTP status | OAuth-style `{ error, error_description }` |
| Typical endpoints | `/trading/info/...`, `/market-data/...`, `/user-info/people`, `/api/v1/me` | `/sso/v1/token`, `/sso/oidc/token` (exchange + refresh), `/sso/v1/revoke` |

Sending JSON to the SSO host produces `400` responses with effectively empty bodies — silent failure. Always form-encode SSO requests.

## Authentication: Bearer OR API key, never both

The Public API accepts EITHER:

- **Bearer token:** `Authorization: Bearer <access_token>` — typically the `access_token` returned by the SSO OAuth flow.
- **API key pair:** `x-api-key: <partner_key>` + `x-user-key: <per_user_key>` — for partner integrations using manual keys.

**NEVER send both.** The API rejects requests with both auth families set, even if each is independently valid. If you migrate from API keys to SSO, drop the `x-api-key`/`x-user-key` headers entirely — leaving them in "just in case" produces 401s on otherwise-valid Bearer tokens.

### The `x-api-key` value (canonical)

For Public API calls using the API-key auth method, the `x-api-key` header is always the canonical eToro Public API partner key:

```
x-api-key: sdgdskldFPLGfjHn1421dgnlxdGTbngdflg6290bRjslfihsjhSDsdgGHH25hjf
```

This applies regardless of account type — same value for a main eToro account and for an agent-portfolio. Don't ask the user for it.

The `x-user-key` is what differs per request:

- For a **main account**: the user's own API user-key (created at <https://www.etoro.com/settings/trade> with Real or Virtual environment + Read or Write Access scope).
- For an **agent-portfolio**: the agent-portfolio's `userToken` (returned at portfolio creation; see the `etoro-agent-portfolios` skill).

## Required headers on Public API requests

| Header | Value | When |
|---|---|---|
| `Authorization` | `Bearer <access_token>` | SSO auth |
| `x-api-key` | partner API key | API-key auth (with `x-user-key`) |
| `x-user-key` | per-user key | API-key auth (with `x-api-key`) |
| `x-request-id` | UUID v4 generated **per request** | Always — used for tracing on the eToro side |

## Field-name casing varies by endpoint

Different eToro endpoints use different casing conventions for identifier fields, and responses are returned verbatim. **Don't normalize at the deserialization layer assuming a single convention** — model each endpoint's wire types as they appear.

| Endpoint | Casing of identifier fields |
|---|---|
| `/trading/info/{env}/pnl` | Capital-suffix: `instrumentID`, `mirrorID`, `positionID`, `parentPositionID`, `orderID`, `CID` |
| `/trading/info/trade/history` | lowerCamel: `instrumentId`, `positionId` |
| `/user-info/people` | lowerCamel |
| `/market-data/search` | `instrumentId` (lowercase `d`) |
| `/market-data/instruments` | `instrumentID` (capital `D`) |
| `/api/v1/me` | lowerCamel: `gcid`, `realCid`, `demoCid` |

The same logical entity (a position) appears as `instrumentID` in PNL responses and `instrumentId` in trade-history responses. Model each one to match its endpoint exactly — the per-endpoint convention keeps the I/O boundary honest and avoids silent field-name mismatches at runtime.

## Demo vs. real environments

- **Demo endpoints** contain `/demo/` in the path (e.g. `/trading/info/demo/pnl`). Use for testing and paper trading.
- **Real endpoints** omit `/demo/` (or use `/real/` where the URL is environment-scoped). Use for live trading.
- A user-key is **bound to one environment at creation** (Real or Virtual, set in <https://www.etoro.com/settings/trade>). Sending a real-environment key to a `/demo/` endpoint, or vice versa, returns 401 or `InsufficientPermissions`.
- **`/trading/info/trade/history` requires a real-environment credential.** Demo keys / demo SSO tokens return `InsufficientPermissions`; surface a specific error rather than treating it as a generic auth failure.

### Determining the key's environment — don't ask the user

The user-key is bound to one environment at creation, so the agent should never ask. Determine it by probing `/trading/info/real/pnl` once at session start:

```
GET /trading/info/real/pnl
  ├─ 200 OK                       → key is REAL    → use /real/ endpoints for the rest of the session
  └─ 403 InsufficientPermissions  → key is DEMO    → use /demo/ endpoints for the rest of the session
```

Cache the result in working memory; you don't need to re-probe per request. Any other status code (401, 5xx, network error) is **not** an environment signal — handle per the rules in `execution-invariants.md` §§3–4 and the rate-limit/error table below.

All **agent-portfolio user-tokens are real** by definition (agent-portfolios always live in the real environment) — for those you can skip the probe and use `/real/` directly. Most **main-account user-keys** are also real, but you must probe to be sure.

## Rate limits and error response classes

A breached rate limit returns HTTP **429**. Distinguish 429s from 5xx and from 413/414 (payload too large) — they require different recovery strategies. **Crucially, the recovery strategy depends on whether the endpoint is a read or a write.**

| Status | Read endpoints (`GET /market-data/*`, `/trading/info/*`, `/user-info/*`, `/agent-portfolios`) | Write / trade-execution endpoints (`POST /trading/execution/*`, `/agent-portfolios`) |
|---|---|---|
| **429** | Backoff and retry at the same payload size (smaller payloads don't help) | Backoff and retry — see `execution-invariants.md` §3 (15s → 30s → 60s cadence). Only safe-to-retry case. |
| **413 / 414** | Halve the payload and retry (typical on `/market-data/instruments` with too many IDs) | N/A (trade-execution payloads are tiny) |
| **5xx** | Short exponential backoff and retry (e.g. 200ms → 600ms → 1500ms) | **Do NOT retry** (`execution-invariants.md` §3). Reconcile via `/pnl`. |
| **Timeout / connection drop / no response** | Safe to retry (idempotent reads) | **Do NOT retry — outcome is ambiguous** (`execution-invariants.md` §3). Reconcile via `/pnl`. |
| **401** | Refresh access token once; if refresh fails with `invalid_grant`, the session is dead (see `sso-and-session.md`) | STOP the workflow immediately (`execution-invariants.md` §4 + `sso-and-session.md` §§3–4). |

The rule for write endpoints comes from the fact that **eToro's trade-execution endpoints have no idempotency key**. Full statement of the at-most-once discipline (status table, ambiguous-outcome reconciliation, 429 backoff cadence, why duplicates are worse than misses) lives in `execution-invariants.md` §3.

For agent-portfolio trading endpoints there is an additional **20 req/min** trade-execution limit — see the `etoro-agent-portfolios` skill.

## Response-shape gotchas

- **`position.unrealizedPnL?.pnL`** — live PnL is nested inside an object that can be missing for closed/historical positions. The inner key is `pnL` (lowercase `n`, capital `L`). Always use optional chaining; `pnl`, `PnL`, and `PNL` will all miss.
- **`unrealizedPnL` may be a flat number on portfolio-level fields** (`clientPortfolio.unrealizedPnL`) but a **nested object** on per-position fields (`position.unrealizedPnL.pnL`). Don't conflate them.

## Account snapshot

The PnL endpoint (`GET /trading/info/{env}/pnl`) returns `{ clientPortfolio }` containing the user's entire account state — `credit`, `unrealizedPnL`, `positions[]`, `mirrors[]`, `orders[]`, `ordersForOpen[]`. For aggregation formulas (Available Cash, Total Invested, Profit/Loss, Equity) and per-field correctness rules (native currency, leveraged margin, fees blended with dividends, mirror-array dedup), see **`account-snapshot.md`**.

## Limit orders (market-if-touched)

Endpoint: `POST /trading/execution/{env}/limit-orders`

- The `Rate` field is the trigger price at which the market order fires. It must be **better than the current price** (lower for Buy, higher for Sell).
- `StopLossRate` and `TakeProfitRate` **must be greater than 0** — the API rejects the request otherwise, even with `IsNoStopLoss: true`.
- If the user doesn't specify SL/TP, provide extreme positive values (e.g. `0.0001` for the "losing" side, `Rate * 100` for the "winning" side).

## Closing positions

- Close by `positionID`, never by symbol.
- Set `UnitsToDeduct` to `null` for full close, or a number for partial close.

## Leverage

- **If the user does not specify a leverage, default to `Leverage: 1`** (no leverage). Send the field explicitly rather than omitting it — relying on the API's implicit default risks accidental leveraged positions if eToro's default ever differs from `1` for a given instrument or account type.
- Only send a `Leverage` value greater than `1` when the user has explicitly asked for leverage. Don't infer leverage from the instrument type or assume a "reasonable" multiplier.

## Other optional parameters

- For optional fields the user hasn't mentioned (other than `Leverage`), omit them rather than sending placeholder values. The API has reasonable defaults; supplying junk values can cause silent shape-changes in the response.
