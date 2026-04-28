# SSO Identity & Session Handling

Reference for the `etoro-trading-assistant` skill. Covers what the runtime agent needs to know about identity and session lifecycle. The full SSO/OAuth implementation (auth-code grant, PKCE, token exchange, atomic refresh-token rotation, callback handlers) is **developer-side** — the runtime agent receives and uses the resulting access token, it doesn't manage the OAuth flow itself.

## 1. Resolve identity via `/api/v1/me`

The agent's authenticated session has an access token. Resolve identity by calling:

```
GET /api/v1/me
```

Response shape:

```json
{ "gcid": <number>, "realCid": <number>, "demoCid": <number> }
```

**All three are stable, environment-spanning identifiers in a 1:1 relationship** — given any one of them you can derive the other two. None of them rotates. The choice between them is about **role**, not safety:

- **`gcid`** — global, environment-independent identity. Use as the canonical reference to "this user" across real and demo.
- **`realCid`** — the user's Customer ID (CID) for the real environment. **Whenever an eToro endpoint asks for a `CID` or `cidList`, it means `realCid`** (or `demoCid` in demo) — it does **not** mean `gcid`.
- **`demoCid`** — the equivalent for the demo environment.

## 2. Pass `realCid`, never `gcid`, to `/user-info/people` (and similar CID-taking endpoints)

```
GET /user-info/people?cidList=<realCid>
```

`gcid` and `realCid` are different *numeric values* for the same user — they live in different ID namespaces. The endpoint accepts any number structurally (no validation error fires), so passing a `gcid` will silently resolve to a *different* user whose CID happens to numerically equal that `gcid` value. **This is a hard-to-spot data-leak bug** — the wrong user's profile comes back, with no error.

Always pass `realCid` (or `demoCid` in demo) to anything that takes a `CID` / `cidList` parameter.

## 3. Handle 401 — branch by auth mode, then STOP

A 401 from the Public API means the credentials weren't accepted. The recovery path depends on which auth mode the agent is using. **In both cases, stop the current workflow immediately** — don't keep firing requests, and never summarize work that didn't happen (see §4).

### 3.a — API-key auth (`x-api-key` + `x-user-key`) — the most common runtime mode

If you're authenticating with the API-key pair (which is the case for any agent operating on a user-provided key — main account or agent-portfolio `userToken`), a 401 typically arrives as:

```json
{ "errorCode": "Unauthorized", "errorMessage": "Unauthorized" }
```

This means at least one of the two keys is wrong. Since `x-api-key` is the canonical Public API partner key (hardcoded — see `api-conventions.md`) and never changes, the failure is in the `x-user-key`:

- **The user revoked their API key** in eToro settings (most common, especially if the key was working earlier in the session).
- **The user provided a wrong key from the start** (less common; usually fails immediately on the first call).
- **For agent-portfolios:** the agent-portfolio's `userToken` was revoked or invalidated. The remediation is the same — get a fresh `userToken`.

**There is no refresh mechanism for API-key auth.** You cannot call a refresh endpoint to get a new key. The user has to issue a fresh key (or, for agent-portfolios, may need to create a new agent-portfolio if the original `userToken` is unrecoverable) and provide it to you.

**What to do:**

1. **Stop the current workflow immediately.** Don't keep firing requests; they'll all fail with the same 401.
2. **Tell the user clearly** that their API key is no longer valid (likely revoked) and ask them to:
   - Verify their key at <https://www.etoro.com/settings/trade> (for main-account keys).
   - Issue a new key if needed.
   - Provide the new key to you. (For agent-portfolios where the `userToken` is lost: see the `etoro-agent-portfolios` skill — they may need to create a new agent-portfolio.)
3. **Don't retry the failing call.** Repeated 401s won't change anything; they just confuse the user.

### 3.b — SSO / Bearer-token auth

If you're authenticating with `Authorization: Bearer <access_token>`:

1. Refresh the access token via the SSO token endpoint.
2. If the refresh succeeds, retry the original call once with the new token.
3. If the refresh itself returns `400 invalid_grant`, the session is **dead** — eToro has revoked it server-side, independently of any rotation event. **No amount of retrying will fix this.** Surface a "Reconnect to eToro" message (see §5) and stop.

The biggest UX bug in SSO mode is a generic "Retry" button that calls refresh again and again, racing the user's clicks. Don't do that.

## 4. CRITICAL — never hallucinate trades that didn't happen

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

This same discipline applies regardless of auth mode (API-key or SSO) — the 401 means the credentials are bad; the partial execution and recovery messaging are independent of how the agent will *eventually* re-authenticate.

## 5. Surface a dedicated "Reconnect to eToro" message (SSO mode only)

When you've detected a dead SSO session (per §3.b), tell the user clearly:

> "Your eToro session has expired and needs to be reconnected. Please sign in again."

Then stop the current operation. Do **not** offer a generic "Retry" — neither retry nor refresh will help; only a fresh OAuth flow restores the session.

After the user re-authenticates, the developer-side callback writes a fresh access + refresh token pair against the same `gcid`-keyed user record, and the agent can resume. (For API-key mode, the equivalent is the user issuing a new key per §3.a.)

## 6. Don't unify session-expired with rate-limited

Different error states with completely different recovery paths — keep them distinct in your messaging:

| Symptom | Likely cause | Recovery |
|---|---|---|
| `401 Unauthorized` on API-key auth | User revoked their `x-user-key` | Stop; ask the user for a new key (no refresh possible). |
| `400 invalid_grant` on SSO refresh | Refresh token revoked | "Reconnect to eToro" — re-auth required |
| `429` on a Public API call | Rate limit | Back off and retry (15s → 30s → 60s) |
| `5xx` | Transient eToro-side error | Short backoff, retry |

Showing the same "Something went wrong, retry" UI for all of these traps users in the wrong recovery mode. The first two need credential action; the third needs time; the fourth usually self-heals on retry.

## 7. Don't auto-revoke unless certain (SSO mode)

If the agent has a way to call `/sso/v1/revoke` (e.g. on user-initiated logout), make it **best-effort** — calling revoke with an already-revoked refresh token returns 400 and pollutes the error logs. Wrap it in a try/catch and ignore failures.
