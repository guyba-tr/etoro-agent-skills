# eToro Instrument Resolution

Reference for the `etoro-trading-assistant` skill. eToro identifies instruments (stocks, ETFs, crypto, commodities, indices, currencies) via numeric `instrumentID` values. Most agent operations that touch a specific instrument need this ID — trade execution, metadata fetch, live rate lookup.

> **User CIDs:** the runtime agent rarely needs to maintain a username↔CID mapping. Most Public-API user endpoints accept username directly (`/user-info/people?usernames=<username>`, `/user-info/people/{username}/portfolio/live`, etc.). Resolve user identifiers ad hoc when needed — no caching pattern required at the skill level. For the signed-in user, see §3 below.

## 1. Instrument IDs are stable — cache resolved IDs within a session

eToro IDs aren't expected to change, so any mapping you resolve stays valid indefinitely. But the runtime agent's storage situation is different from a backend service's, so adapt the caching pattern accordingly:

- **Within the current conversation:** cache resolved IDs in working memory. If the user mentions AAPL, resolve it once and reuse for the rest of the session — don't re-call `/market-data/search` on every reference.
- **Across conversations / sessions:** defer to whatever the host runtime provides. Some agent runtimes give skills a persistence layer (key-value store, structured conversation state, vector store); others don't. If the host has caching at the tool layer (e.g. an `etoro_resolve_symbol` tool that hits a server-side cache), prefer it. If neither is available, accept the per-session resolution cost.
- **Don't try to maintain a static hardcoded list** in the skill or in the agent's prompt. The agent typically serves one user per conversation with a small set of symbols; a baked-in catalog is unnecessary and a maintenance liability.

The resolution cost in this context is small — typically a handful of unknown symbols per session. The rate-limit concern that motivates aggressive caching in backend services (many users hitting search for the same symbol concurrently) doesn't manifest the same way for a one-user-per-conversation runtime agent.

## 2. Live-resolve unknown symbols

```
GET /market-data/search?internalSymbolFull=<SYMBOL>
```

- The response array `items[]` may contain multiple matches (e.g. `MSFT`, `MSFT.RTH`, `MSFT.EUR`). Confirm `internalSymbolFull` matches exactly before using the result.
- The search endpoint returns `instrumentId` (lowercase `d`). Most other endpoints use `instrumentID` (capital `D`) — don't accidentally model the search response with the wrong casing. See `api-conventions.md` for the full casing table.
- Once resolved, **cache the result in working memory** for the rest of the conversation so the next reference hits the cache.

## 3. The signed-in user is special

For the *currently signed-in* user, get identity via:

```
GET /api/v1/me
```

Returns `{ gcid, realCid, demoCid }` — see **`sso-and-session.md`** for which CID to use where, the `cidList` vs `gcid` data-leak trap, and why `gcid` is the recommended primary key for any user record.

## 4. Enrich instruments with metadata after ID resolution

Once you have an `instrumentID` (or batch of them), fetch display data:

```
GET /market-data/instruments?instrumentIds=<id1>,<id2>,...
```

- Send IDs in batches of **25–50**; this endpoint rejects large batches with HTTP 413/414. On 413/414, halve the batch size and retry.
- The response uses `instrumentID` (capital `D`) and includes display name, full symbol, exchange, instrument type, and image variants.

### Image variants

Each instrument's `images[]` array is **unsorted** — don't blindly use `images[0]`. The variants typically include:

- A vector "card" SVG carrying `backgroundColor` + `textColor` metadata (best for tile-style UI).
- Several PNG rasters at different widths (50px, 90px, 150px, …).
- Sometimes legacy formats.

Selection logic:

1. Prefer the SVG "card" variant if present.
2. Otherwise pick the largest-width PNG.
3. Fall back to `images[0]` only as a last resort.

## 5. Never guess an ID

If you don't have a verified mapping for a symbol, resolve it via `/market-data/search`. **Never** make up an ID, transform a number, or hash a string — eToro's IDs are dense small integers and you will silently hit a real but wrong instrument.
