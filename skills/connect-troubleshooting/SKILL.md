---
name: connect-troubleshooting
description: Diagnose and fix failures anywhere in the ab0t Connect mesh — the unified error reference and debugging playbook across every surface (connections, proxy, UTS tools, webhooks, websockets, auth). Use when a call failed and you need to know why, or you see any of 401 / 403 / 404 / 409 / 422 / 429 / 502, "unauthorized", "forbidden", "connection not found", "already linked", "validation failed", "rate limited", "Retry-After", "X-Integration-Rate-Limit", "backoff", "webhook not verifying", "delivery failed/timed out", "websocket closed 4001/4003/4500", "missing_scopes", "wrong auth path", "audience mismatch", "not logged in". Also use to "debug a failed call", "why did my request fail", "isolate mesh vs provider", "troubleshoot an error", read X-Request-ID for support, or look up what a status / close code means and how to fix it. One-stop diagnostic — start here, then jump to the surface skill.
---

# connect-troubleshooting — diagnose & fix any Connect failure

> CLI is `connect`. Default host
> `connect.service.ab0t.com`.

A call failed. This skill turns *what you saw* into *what to do* across the whole
mesh. Work the flow top to bottom: identify the surface, read the code, apply the
fix. The full code-by-code table is in
[references/error-reference.md](references/error-reference.md); rate limits and
backoff are in [references/rate-limits.md](references/rate-limits.md).

## Fast diagnostic flow

**1. Which surface failed?**

| You were doing… | Surface | Path shape |
|---|---|---|
| Creating / listing / revoking a connection | **connections** | `/connections/…` |
| Calling a provider's native API | **proxy** | `/proxy/{connection_id}/…` |
| Running a normalized tool | **tools (UTS)** | `/uts/v1/tools/…` |
| Receiving HTTP-pushed events | **webhooks** | `/v2/webhooks/v1/…` + your receiver |
| Streaming live events | **websockets** | `wss://…/v2/webhooks/v1/ws` |
| Logging in / API key / profile | **auth** | any call's `Authorization` / `X-API-Key` |

**2. Read the status (or WS close) code.** Same number can mean different things
per surface — the surface tells you which row to read.

| Code | One-line read (varies by surface — see full table) |
|---|---|
| `401` | Mesh auth failed (expired/wrong-audience token or key). **Exception:** a `401` carrying a *provider* body on a proxy/tool call = the provider rejected its injected credential. |
| `403` | Forbidden — almost always a `connection_id` that belongs to **another org** (cross-tenant blocked). |
| `404` | Not found — unknown service id, `connection_id`, or `category/tool`. |
| `409` | That account/workspace is **already linked** to a connection in your org. |
| `422` | Validation — bad request body, bad envelope, or `missing_scopes` on a subscription. |
| `429` | **Rate limited.** Read `Retry-After` / `X-Integration-Rate-Limit`, back off, retry. |
| `502` | Mesh couldn't reach the provider (network). Retry after the hint. |
| `4xx/5xx` w/ provider body (proxy) | The **provider's own** error, passed through — debug against the provider, not ab0t. |
| WS `4001` | No `?token=` on the socket — add one. |
| WS `4003` | Bad/expired token or out-of-scope tenant — refresh / fix scope. |
| WS `4500` | Mesh internal error — back off and reconnect. |

**3. Apply the fix** from the row in
[references/error-reference.md](references/error-reference.md), then re-run.

## The 30-second debug checklist

Run these in order; each one isolates a layer:

1. **Is the connection active?** `connect info <connection_id>` (or
 `connect connections show <connection_id>`) — confirm `status: active`. A
 revoked/expired connection causes proxy `400`, tool `400`, and proxy-`401`
 (provider-body) failures.
2. **Is YOUR auth good?** `connect auth whoami` (or `connect status`). Confirms
 the token/key reaches the mesh and you're on the **expected profile**
 (`connect profile show`) — a "not logged in" / `401` is often the wrong
 profile, not a dead token.
3. **Is the `connection_id` yours?** A `403` means it belongs to another org.
 `connect connections list` — only ids you see are actionable.
4. **Capture `X-Request-ID`** from the response headers. It correlates the call
 in mesh logs/audit — quote it to support for any mesh-side failure.
5. **Isolate mesh vs provider.** Re-run the *same* call as a bare
 `connect call <conn> GET /…` proxy request. If the proxy returns the
 provider's own `4xx/5xx` body, the fault is upstream (fix at the provider). If
 the mesh rejects it before reaching the provider (`400`/`403`/`404`/`422`),
 the fault is your request or connection.

## Top pitfalls

- **Wrong auth path when connecting.** Api-key path on an OAuth service (or vice
 versa) → `400`. Use the path matching `connect services show <svc>`.
- **Re-consenting when a connection already exists** → `409`. List first, reuse
 the `connection_id` — never re-create.
- **Smuggling credential keys** (`access_token`, `tenant_id`, …) into
 `additional_config` → `422`. Those keys are reserved.
- **Treating a passthrough provider error as a mesh bug.** A `4xx/5xx` with an
 upstream body on `/proxy/…` is the *provider's* error — debug it there.
- **Subscribing to event types the connection's scopes don't cover** → `422
 missing_scopes`. Reconnect with the missing scopes, then retry.
- **Webhook receiver not verifying / acking slowly.** Sign the **raw body**,
 return **2xx within a few seconds**, dedupe on `X-Webhook-Id` — slow handlers
 time out → retries → duplicates.
- **Hot-looping a websocket on `4001`/`4003`.** Those are auth faults — fix the
 token first; only reconnect-with-backoff on `1000` / `4500`.
- **Ignoring `Retry-After` on `429`.** Honor the header and use exponential
 backoff — see [references/rate-limits.md](references/rate-limits.md).

## References

| File | When to read |
|---|---|
| [references/error-reference.md](references/error-reference.md) | The consolidated table: every status & WS close code × surface → meaning → fix, grouped by connections / proxy / tools / webhooks+ws / auth. |
| [references/rate-limits.md](references/rate-limits.md) | How mesh rate limiting works, the `X-Integration-Rate-Limit` / `Retry-After` headers, `429` handling, and exponential-backoff guidance. |

Once you know the surface, the matching skill (`connect-connections`,
`connect-proxy`, `connect-tools`, `connect-realtime`, `connect-auth`) has the full
how-to. The live `GET /openapi.json` is the authoritative contract.
