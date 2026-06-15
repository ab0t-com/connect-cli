---
name: connect-proxy
description: Call a connected provider's OWN native API through the ab0t connect mesh's authenticated proxy passthrough — the `/proxy/{connection_id}/<path>` surface. Use when you want to proxy a request to a provider, call the provider's API without handling its auth, hit a raw provider endpoint, GET/POST/PUT/PATCH/DELETE through a connection, reach an endpoint that has no normalized UTS tool, do a one-off or fallback provider call, read X-Integration-* response headers, or debug 401/403/404/409 on a `/proxy/` call. Keywords — proxy a request, call the provider's API, passthrough, raw provider API, /proxy/{connection_id}, X-Integration headers, universal fallback for any provider endpoint.
---

# connect-proxy — call a provider's own API through the mesh

> CLI is `connect`. Default host
> `connect.service.ab0t.com`.

The proxy is a **transparent passthrough**: you address the provider's *native*
API path, the mesh injects that connection's upstream credential and any
provider-specific headers **server-side**, runs the call, and hands you the
provider's raw response back. You never see or hold the provider token.

```
{METHOD} {api_url}/proxy/{connection_id}/{provider_native_path}
```

- `{connection_id}` selects one account of one provider and injects its credential.
- `{provider_native_path}` is the upstream path verbatim (e.g.
 `/calendar/v3/users/me/calendarList`, `/v3/marketing/contacts`).
- **The HTTP method on YOUR call is the method used upstream.** GET proxies a
 GET, POST proxies a POST. All of GET/POST/PUT/PATCH/DELETE (plus HEAD/OPTIONS)
 pass through.
- Query string and request body pass through unchanged.

## What the proxy does server-side

You send a plain request with **no provider auth**. The mesh:

1. **Injects the upstream credential** — the stored OAuth access token or API
 key for that connection. It strips/replaces your `Authorization` header; do
 not send the provider's token yourself.
2. **Injects provider-specific headers** — e.g. Xero's `xero-tenant-id`, Google
 Ads' `developer-token`, Meta's `appsecret_proof`. You never compute or attach
 these.
3. **Refreshes OAuth tokens on demand** — expired access tokens are refreshed
 transparently before the call.
4. **Applies rate limiting and audits** the call.

This is why the proxy is the **universal fallback**: any provider endpoint that
isn't wrapped as a normalized UTS tool is still reachable here, with auth fully
handled for you.

## CLI

```bash
# call = convenience verb (METHOD as a positional arg)
connect call <connection_id> GET /user/repos?per_page=100
connect call <connection_id> GET '/calendar/v3/users/me/calendarList'

# proxy = METHOD first, connection + path joined, body via --data
connect proxy POST <connection_id>/calendar/v3/calendars/primary/events \
 --data '{"summary":"Standup",
 "start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}'

# connection metadata — NO upstream call (service, scopes, status, account)
connect info <connection_id>

connect call <connection_id> GET /v3/x -o json # machine-readable for scripting
```

Discover the connection first: `connect connections list` → pick the
`connection_id` → `connect info <connection_id>` to confirm it is `active`.

## Raw HTTP (curl)

Authenticate to the mesh with your ab0t key (`X-API-Key: ab0t_sk_live_…`). That
is the ONLY auth you send — the provider credential is injected for you.

```bash
URL=https://connect.service.ab0t.com
KEY=ab0t_sk_live_..

# GET passthrough (GitHub: list the connected user's repos)
curl "$URL/proxy/$CONN/user/repos?per_page=100" -H "X-API-Key: $KEY"

# POST passthrough (Google Calendar: create an event)
curl -X POST "$URL/proxy/$CONN/calendar/v3/calendars/primary/events" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"summary":"Sync",
 "start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}'

# Connection metadata only (no upstream call)
curl "$URL/proxy/$CONN/info" -H "X-API-Key: $KEY"
```

The body you get back IS the upstream provider's response — its status code, its
headers, its body, passed through.

### Provider examples (auth injected for you)

```bash
# Xero — xero-tenant-id header injected server-side
curl "$URL/proxy/$CONN/api.xro/2.0/Invoices" -H "X-API-Key: $KEY"
connect call $CONN GET /api.xro/2.0/Invoices

# Google Ads — developer-token injected server-side
connect call $CONN POST /v17/customers/1234567890/googleAds:search \
 --data '{"query":"SELECT campaign.id FROM campaign"}'

# Meta (Graph) — appsecret_proof injected server-side
connect call $CONN GET '/v20.0/me/adaccounts?fields=name,account_status'
```

In every case you write only the native path + body. The tenant id, developer
token, and appsecret proof are added by the mesh — they never appear in your call.

## Response headers

The mesh adds these on top of the provider's own response headers:

- `X-Integration-Service` — the provider id behind this connection.
- `X-Integration-Response-Time` — round-trip time, e.g. `142.5ms`.
- `X-Request-ID` — correlates the call in mesh logs/audit.
- A `X-Integration-Rate-Limit` / `X-Integration-Cost` header may also appear
 when the mesh tracks rate-limit state or cost for the call.

Hop-by-hop and auth headers are filtered out in both directions.

## Proxy vs UTS tools — which to reach for

| | Proxy (`/proxy/…`) | UTS tools (`/uts/v1/tools/…`) |
|---|---|---|
| Path | provider's **native** API path | normalized `category/tool` |
| Shape | provider's raw request/response | unified, validated envelope |
| Discovery | provider's own docs | `connect tools catalog` |
| Use when | you know the provider's API, or need an endpoint no tool wraps | you want one consistent schema across providers |

Rule of thumb: **prefer a UTS tool when one exists** (validation, structured
errors). **Reach for the proxy** for full provider coverage, one-off endpoints,
or anything the tool catalog doesn't expose. The proxy is the universal fallback.

## Errors

| Status | Meaning | Fix |
|---|---|---|
| `404` | `connection_id` doesn't exist (or isn't yours) | `connect connections list` |
| `403` | Connection belongs to another tenant — cross-tenant blocked | use your own connection |
| `400` | Connection is not active (revoked / not yet usable) | re-connect the provider |
| `4xx`/`5xx` with an upstream body | The **provider's** own error, passed through | debug against the provider, not ab0t |
| `502` | Mesh couldn't reach the provider (network) | retry after the `retry_after` hint |

A `401` carrying a provider body is the provider rejecting the injected
credential (e.g. OAuth no longer refreshable) — re-do the connection's consent.

## Patterns

- **Pagination and rate limits are the provider's.** The proxy does not
 auto-paginate; honor the provider's `Link`/cursor/`Retry-After`.
- **Multiple accounts** of one provider = multiple connections. Pick the right
 `connection_id`.
- **Confirm before calling:** `connect info <connection_id>` to verify `active`
 and see the scopes/account, without spending an upstream call.

More detail and a longer error/provider table:
[`references/proxy.md`](references/proxy.md).
