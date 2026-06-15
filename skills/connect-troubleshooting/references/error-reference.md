# Connect mesh — unified error reference

> CLI is `connect`. Default host
> `connect.service.ab0t.com`.

Every status code and websocket close code across every Connect surface, with the
meaning **on that surface** and the fix. The [`SKILL.md`](./SKILL.md) has the
diagnostic flow and the debug checklist; this file is the lookup table.

## Contents
- [Connections](#connections)
- [Proxy (provider passthrough)](#proxy-provider-passthrough)
- [Tools (UTS)](#tools-uts)
- [Webhooks + Websockets](#webhooks--websockets)
- [Auth (your credential to the mesh)](#auth-your-credential-to-the-mesh)
- [Cross-surface notes](#cross-surface-notes)

---

## Connections

The `/connections/…` surface — creating, listing, revoking, and using a
connection.

| Code | Meaning | Fix |
|---|---|---|
| `400` | Wrong auth path for the service — api-key path on an OAuth service, or vice versa | Use the path matching `connect services show <svc>` |
| `401` (on a later call) | Stored OAuth credential expired and isn't refreshable | `connect connections show <conn>`; re-do OAuth, or update the api-key connection |
| `403` | That `connection_id` belongs to another org | `connect connections list` — act only on your own |
| `404` | Service id unknown, or `connection_id` not found | Check `connect services list`; check `connect connections list` |
| `409` | Workspace/account already linked to another connection in this org | **Reuse** the existing connection — don't re-create |
| `422` | Bad request body — missing `api_key`, a reserved credential key in `additional_config`, or a bad `redirect_uri` | Fix the body; keep credential/routing keys (`access_token`, `refresh_token`, `tenant_id`) out of `additional_config` |

---

## Proxy (provider passthrough)

The `/proxy/{connection_id}/<path>` surface. **Key distinction:** a code with a
*mesh* body is the mesh rejecting your call; a code (incl. `4xx/5xx`) with an
*upstream provider* body is the provider's own response passed through.

| Code | Meaning | Fix |
|---|---|---|
| `400` | Connection is not active (revoked / not yet usable) | Re-connect the provider; `connect info <conn>` to confirm `active` |
| `401` (provider body) | The provider rejected the injected credential (OAuth no longer refreshable) | Re-do the connection's consent (re-OAuth) |
| `403` | Connection belongs to another tenant — cross-tenant blocked | Use your own connection |
| `404` | `connection_id` doesn't exist (or isn't yours) | `connect connections list` |
| `429` | Rate limited (mesh or provider) | Honor `Retry-After` / `X-Integration-Rate-Limit`; back off — see [rate-limits.md](rate-limits.md) |
| `502` | Mesh couldn't reach the provider (network) | Retry after the `retry_after` hint |
| `4xx`/`5xx` w/ upstream body | The **provider's own** error, passed through verbatim | Debug against the provider, not ab0t — fix the request the provider rejected |

Provider pagination and rate limits are the **provider's** — honor its
`Link`/cursor and `Retry-After`; the proxy does not auto-paginate. Read
`X-Integration-Service`, `X-Integration-Response-Time`, and `X-Request-ID` from
the response headers when diagnosing.

---

## Tools (UTS)

The `/uts/v1/tools/…/execute` surface. Two layers of errors: the **execute-call**
status (below) and each tool's own **`errors` catalog** (provider-level codes).

| Code | Meaning | Fix |
|---|---|---|
| `400` | Input failed schema validation, OR the connection's service doesn't match the tool's category | Run `connect tools validate …` first; use a connection of the right service |
| `403` | Tenant boundary — that `connection_id` belongs to another org | `connect connections list` to find your own |
| `404` | Unknown tool, or missing connection | `connect tools catalog`; check the exact `category/tool` |
| `422` | Missing/ill-formed `connection_id` or `input` in the request body | Check the envelope shape (`{connection_id, input:{path_params,query_params,body,headers}}`) |

**Tool-catalog errors** are typed, not free-text. Each carries `status`, `code`,
`category`, `retryable`, and `retry_after`/resolution — e.g.
`{status:403, code:"NOT_IN_CHANNEL", retryable:false}` or
`{status:429, code:"RATE_LIMITED", retryable:true, retry_after:60}`. Read them via
`connect tools show <category>/<tool>` and branch on `code` + `retryable` instead
of parsing text.

---

## Webhooks + Websockets

The `/v2/webhooks/v1/…` surface (subscriptions, deliveries, the inbound-events
audit) and the `wss://…/v2/webhooks/v1/ws` realtime stream.

### Subscriptions & deliveries (HTTP)

| Code | Meaning | Fix |
|---|---|---|
| `422 missing_scopes` | Chosen `event_types` need OAuth scopes the connection was never granted | Reconnect the connection **with** the missing scopes, then retry the subscribe |
| `400` (on retry) | You tried to retry a delivery that already succeeded | Only `failed` / `timeout` deliveries are retryable |
| delivery `failed` | Your receiver returned non-2xx, or a connection error | Fix the receiver; retried automatically with exponential backoff; `connect webhooks deliveries retry <id>` to re-queue |
| delivery `timeout` | Receiver didn't return 2xx within a few seconds | Ack **2xx fast**, process off the request path; retried with backoff |

**Receiver-side (your endpoint):** verify the `X-Webhook-Signature` HMAC over the
**raw body**, reject on signature mismatch or `t` skew > ~300s, dedupe on
`X-Webhook-Id` (deliveries are at-least-once), return **2xx within a few seconds**.

### Websocket close codes

| Close | Meaning | Fix |
|---|---|---|
| `1000` | Normal close | Reconnect with backoff if you still want the stream |
| `4001` | **Auth required** — no token supplied | Add `?token=<jwt_or_api_key>` to the `wss` URL |
| `4003` | **Auth failed** — bad/expired token, or out-of-scope tenant | Refresh the token; check tenant scope; **don't** hot-loop |
| `4500` | Mesh internal error | Back off and reconnect |

Reconnect with exponential backoff on `1000` / `4500`. **Never** hot-loop on
`4001` / `4003` — fix the token first. A `{"type":"error","data":{"code",
"message"}}` frame reports an in-stream problem without closing the socket.

---

## Auth (your credential to the mesh)

Every authenticated call carries `Authorization: Bearer <token>` or
`X-API-Key: <key>`. These errors are about **your** credential, independent of any
connection.

| Code | Meaning | Fix |
|---|---|---|
| `401 unauthorized` | Token expired, or wrong audience for this host | Re-`connect login` (human) or rotate + re-export the API key; confirm `connect auth whoami` |
| "not logged in" | No credential on the **active profile** | `connect login` (human) or `export CONNECT_API_KEY=…` (agent). Check `connect profile show` — you may be authed on a *different* profile |
| audience mismatch | The token was minted for a different host/environment | Use the right profile (`connect profile use <name>`) or one-shot with `--api-url` / `CONNECT_API_URL` |

The ambient `CONNECT_API_KEY` env var takes precedence over any stored credential
for the active profile — handy for one-shot agent/CI runs.

---

## Cross-surface notes

- **`403` is almost always cross-tenant.** Across connections, proxy, and tools,
 a `403` means the `connection_id` isn't yours. `connect connections list`.
- **`404` is "unknown id".** Service id, `connection_id`, or `category/tool` —
 check the relevant `list`/`catalog`.
- **`401` is ambiguous by body.** A mesh body = *your* credential is bad (auth
 surface). A *provider* body on a proxy/tool call = the injected provider
 credential is bad (re-OAuth the connection).
- **`429` everywhere** → [rate-limits.md](rate-limits.md). Honor `Retry-After` /
 `X-Integration-Rate-Limit` and back off.
- **Always capture `X-Request-ID`** for any mesh-side failure — it's the
 correlation id support needs.
