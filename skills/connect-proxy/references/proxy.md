# Proxy passthrough — reference

> CLI is `connect`. Default host
> `connect.service.ab0t.com`.

Deep reference for the `/proxy/{connection_id}/<path>` surface. The
[`SKILL.md`](./SKILL.md) covers the model and quick examples; this file is the
detail.

## Contents
- [The contract](#the-contract)
- [Method semantics](#method-semantics)
- [What is injected server-side](#what-is-injected-server-side)
- [Headers in and out](#headers-in-and-out)
- [Bodies and content types](#bodies-and-content-types)
- [Per-provider examples](#per-provider-examples)
- [The info endpoint](#the-info-endpoint)
- [Error reference](#error-reference)
- [When NOT to use the proxy](#when-not-to-use-the-proxy)

## The contract

```
{METHOD} {api_url}/proxy/{connection_id}/{provider_native_path}[?query]
```

The proxy forwards to the upstream provider with the connection's credential
injected and tenancy enforced. The response you receive is the upstream
provider's response (status, headers, body) passed back through, plus a few
`X-Integration-*` / `X-Request-ID` headers the mesh adds.

You only ever act on your own org's connections. Wrong account, missing, or
another tenant's connection is rejected server-side before any upstream call.

## Method semantics

Every HTTP method passes through and is used **verbatim upstream**:

| Your call | Upstream |
|---|---|
| `GET /proxy/{conn}/path` | `GET path` on the provider |
| `POST /proxy/{conn}/path` | `POST path` on the provider |
| `PUT` / `PATCH` / `DELETE` | same method on the provider |
| `HEAD` / `OPTIONS` | same method on the provider |

There is no separate "write" surface — the method you choose IS the upstream
method. Request bodies are forwarded on `POST`/`PUT`/`PATCH`.

## What is injected server-side

You send the request with **no provider auth**. The mesh adds, per connection:

- **The upstream credential** — the stored OAuth access token or API key. Your
 own `Authorization` header is stripped and replaced.
- **OAuth refresh on demand** — an expired access token is refreshed before the
 upstream call; you never manage refresh tokens.
- **Provider-specific headers** the provider requires beyond bearer auth, e.g.:
 - Xero — `xero-tenant-id` for the connected organisation.
 - Google Ads — `developer-token`.
 - Meta / Facebook Graph — `appsecret_proof`.
- **Rate-limit mediation and audit** — the call is rate-limited and recorded.

Because auth is fully handled, **any endpoint the provider exposes is reachable**
even when there is no normalized UTS tool for it. The proxy is the universal
fallback for full provider API coverage.

## Headers in and out

- **Outbound (your request → provider):** query string, body, and most headers
 pass through. Hop-by-hop headers (`connection`, `transfer-encoding`, `te`,
 `trailers`, `upgrade`, `proxy-*`) and your `authorization` / `host` /
 `content-length` are removed — the mesh sets the real upstream auth.
- **Inbound (provider response → you):** the provider's status, content-type,
 and body come straight back. The mesh adds:
 - `X-Integration-Service` — provider id behind the connection.
 - `X-Integration-Response-Time` — e.g. `142.5ms`.
 - `X-Request-ID` — correlation id for the call.
 - `X-Integration-Rate-Limit` / `X-Integration-Cost` when the mesh tracks them.
 - `content-encoding` is dropped to avoid double-compression.

## Bodies and content types

- `application/json` bodies pass through as JSON.
- `multipart/form-data` (file uploads) is supported — fields and file parts are
 forwarded.
- Other content types are forwarded as a raw body.

## Per-provider examples

```bash
URL=https://connect.service.ab0t.com
KEY=ab0t_sk_live_..

# GitHub — list the connected user's repos
curl "$URL/proxy/$CONN/user/repos?per_page=100" -H "X-API-Key: $KEY"
connect call $CONN GET /user/repos?per_page=100

# Google Calendar — create an event (POST body forwarded)
curl -X POST "$URL/proxy/$CONN/calendar/v3/calendars/primary/events" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"summary":"Sync","start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}'
connect proxy POST $CONN/calendar/v3/calendars/primary/events \
 --data '{"summary":"Sync","start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}'

# Xero — list invoices (xero-tenant-id injected)
connect call $CONN GET /api.xro/2.0/Invoices

# SendGrid — list marketing contacts (API-key connection)
connect call $CONN GET /v3/marketing/contacts

# Meta Graph — read ad accounts (appsecret_proof injected)
connect call $CONN GET '/v20.0/me/adaccounts?fields=name,account_status'
```

You only write the native path and body in each case. The provider-specific
auth header is added by the mesh and never appears in your request.

## The info endpoint

`GET /proxy/{connection_id}/info` returns connection metadata **without** making
any upstream call — use it to confirm a connection is usable before a real call:

```bash
curl "$URL/proxy/$CONN/info" -H "X-API-Key: $KEY"
connect info $CONN
```

Returns: `service_id`, `service_name`, `scopes`, `status` (look for `active`),
and account info for the connected account.

## Error reference

| Status | Source | Meaning | Action |
|---|---|---|---|
| `404` | mesh | `connection_id` unknown or not yours | `connect connections list` |
| `403` | mesh | Cross-tenant — connection belongs to another org | use your own connection |
| `400` | mesh | Connection is not active (revoked / pending) | re-connect the provider |
| `502` | mesh | Couldn't reach the provider (network) | retry after `retry_after` |
| `401` | provider | Injected credential rejected (OAuth not refreshable) | re-do the connection's consent |
| other `4xx`/`5xx` | provider | The provider's own error, passed through | debug against the provider's docs |

Tell mesh errors from provider errors by the body: a mesh error has an ab0t
`detail`; a provider error carries the provider's own error shape.

## When NOT to use the proxy

- A **normalized UTS tool exists** for the operation — prefer it for input
 validation and structured errors. The proxy is the fallback, not the default.
- You need **auto-pagination** — the proxy does not paginate; it returns one
 upstream response. Loop over the provider's own cursor/`Link` headers yourself.
