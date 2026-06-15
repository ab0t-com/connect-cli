---
name: connect-rest-api
description: >-
 Call the ab0t Connect mesh PUBLIC REST API directly — over raw HTTP from
 Python, JS, Go, or curl — WITHOUT the CLI. Use for a server-side integration
 that talks straight to the endpoints instead of shelling out to
 connect/connect. Triggers: "REST API", "call the API directly", "without
 the CLI", "raw HTTP", "curl", "Python/JS/Go HTTP client", "requests / fetch /
 net/http", "/openapi.json", "Bearer token", "X-API-Key auth", "Authorization
 header", "server-side integration", "headless", "GET /services", "/discovery",
 "/connections", "POST /connections oauth authorize", "connection api-key",
 "/proxy/{connection_id}", "passthrough to the provider API", "/uts/v1/tools",
 "tool execute / validate / batch", "/v2/webhooks/v1 subscriptions /
 deliveries / events", "the public webhook receiver", "wss websocket stream",
 "/health". Covers auth (Bearer JWT or X-API-Key, org-scoped; public discovery
 needs none), every public endpoint group with a curl recipe, and the response
 conventions. The live GET /openapi.json is the source of truth for exact
 request/response shapes.
---

> CLI naming: the Connect CLI is `connect` and is being renamed to
> `connect` (ticket 7f3a) — this skill is the **no-CLI** path, so it matters
> only when you cross-reference CLI docs. Host: `connect.service.ab0t.com`
> is being renamed to `connect.service.ab0t.com`; both resolve to the same mesh.
> Examples below use the current host.

# Calling the Connect mesh REST API directly

The Connect mesh is a plain HTTPS JSON API. Anything the CLI does, you can do
with a raw HTTP client. This skill is for **server-side integrations** that hold
a credential and call the endpoints themselves — no `connect`/`connect` on the
box.

```
base_url = https://connect.service.ab0t.com (→ connect.service.ab0t.com)
```

**Source of truth:** `GET {base_url}/openapi.json` is the authoritative,
always-current contract for every path, parameter, and request/response schema.
Read it at runtime; this skill describes the patterns and the most-used calls,
the spec describes the exact shapes. Per-group curl recipes for every endpoint
are in [references/endpoints.md](references/endpoints.md).

## The model in one paragraph

A **connection** is a stored, encrypted credential for one account of one
provider, owned by your org, identified by an opaque `connection_id`. You create
it once (consent), then **everything you do acts through that `connection_id`** —
proxying the provider's API, running UTS tools, subscribing to events. Tenancy is
enforced server-side: you only ever see and act on your own org's connections.

## Auth

Every authenticated request carries one of these headers:

```
Authorization: Bearer <jwt-or-apikey> # preferred
X-API-Key: <apikey> # also accepted
```

- The token is **either** a mesh **JWT user token** (obtained via the auth mesh,
 short-lived) **or** an **API key** (an `ab0t_sk_live_…` mesh key, headless, no
 browser — ideal for server-side). The mesh tells them apart by token shape; for
 `Authorization: Bearer` you may pass either value.
- Requests are **org-scoped** by the token's identity. The mesh derives your org
 from the credential — you do **not** pass an `org_id` parameter.
- **Public / discovery / health endpoints need no auth** (`/services`,
 `/discovery/*`, `/health`, `/openapi.json`, `/llm.txt`). Acting endpoints
 (`/connections`, `/proxy`, tool execute, webhook subscriptions) require auth.

```bash
BASE=https://connect.service.ab0t.com
KEY=ab0t_sk_live_.. # your mesh API key

# authed call
curl -s "$BASE/connections/" -H "Authorization: Bearer $KEY"
# equivalently
curl -s "$BASE/connections/" -H "X-API-Key: $KEY"
```

Common auth failures: **401** missing/expired/wrong-audience token; **403** the
`connection_id` belongs to another org; **400** wrong connection type for the
operation; **404** connection or tool not found.

## Conventions

- `connection_id` is **opaque** — never parse or construct it; store the string
 the create call returns and pass it back verbatim.
- All timestamps are **ISO-8601 UTC** (e.g. `2026-09-01T10:00:00Z`).
- Request and response bodies are JSON; send `Content-Type: application/json`.
- Lists are paginated (`page`/`page_size`, plus `total` / `has_more` / cursors on
 some endpoints) — check the spec per endpoint.
- Trailing slashes follow the spec exactly (`/connections/`, `/services/`,
 `/v2/webhooks/v1/subscriptions/`). Use the path as written.

## Endpoint groups

| Group | Base path | Auth | What it does |
| --- | --- | --- | --- |
| Discovery | `/services`, `/discovery/*` | none | browse connectable providers, their tools, OpenAPI, events |
| Connections | `/connections..` | yes | create (API-key / OAuth), list, get, delete |
| Proxy | `/proxy/{connection_id}/<path>` | yes | transparent passthrough to the provider's own API |
| UTS tools | `/uts/v1/tools..` | yes (execute) | normalized, schema-validated operations; validate / execute / batch |
| Webhooks v2 | `/v2/webhooks/v1/..` | yes (+ public receiver) | subscriptions, the public receiver, deliveries, events |
| Websockets | `wss ../v2/webhooks/v1/ws?token=` | token in query | live realtime event stream |
| Health / meta | `/health`, `/openapi.json`, `/llm.txt`, `/status` | none | liveness + the contract |

Full per-group curl recipes (with request bodies and response fields):
**[references/endpoints.md](references/endpoints.md)**.

## The most-used calls

**Discover providers (no auth):**

```bash
curl -s "$BASE/services/" # connectable provider catalog
curl -s "$BASE/services/github" # one provider: auth type, scopes, webhooks
curl -s "$BASE/discovery/services" # discovery manifest variant
```

**Create a connection — API-key provider (fully headless):**

```bash
curl -s -X POST "$BASE/connections/sendgrid/api-key" \
 -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
 -d '{"api_key":"SG.xxxx","name":"prod"}'
# -> { "connection_id": "..", .. } ← act through connection_id
```

**Create a connection — OAuth provider (one-time human consent):**

```bash
# 1) start the flow — returns an authorization_url a human opens once
curl -s -X POST "$BASE/connections/github/oauth/authorize" \
 -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
 -d '{"redirect_uri":"https://app.example.com/callback","scopes":["repo"]}'
# -> { "authorization_url": "..", "state": "..", "expires_in": 300 }

# 2) after the provider redirects back with ?code&state, complete it
curl -s -X POST "$BASE/connections/github/oauth/callback" \
 -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
 -d '{"code":"<code>","state":"<state>","redirect_uri":"https://app.example.com/callback"}'
# -> { "connection_id": "..", "status": "active", .. }
```

OAuth is a **consent flow only** — you cannot inject `access_token` /
`refresh_token` via the api-key endpoint (reserved keys are stripped
server-side). List your connections with `GET /connections/`; reuse an existing
valid `connection_id` rather than re-consenting.

**Proxy a call to the provider's own API** (the method on YOUR call is the method
used upstream; write the provider's native path):

```bash
curl -s "$BASE/proxy/<connection_id>/user/repos?per_page=100" \
 -H "Authorization: Bearer $KEY"

curl -s -X POST "$BASE/proxy/<connection_id>/calendar/v3/calendars/primary/events" \
 -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
 -d '{"summary":"Standup","start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}'
```

The mesh injects the stored upstream credential, refreshes OAuth tokens on
demand, applies rate limits, and audits the call. Response headers may include
`X-Integration-Service`, `X-Integration-Rate-Limit`, `X-Integration-Cost`.

**Execute a UTS tool** (normalized + schema-validated; prefer over raw proxy when
a tool exists). The input envelope is `path_params` / `query_params` / `body`:

```bash
curl -s -X POST "$BASE/uts/v1/tools/google-calendar/create_event/execute" \
 -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"<connection_id>",
 "input":{"path_params":{"calendarId":"primary"},
 "body":{"summary":"Hi",
 "start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}}}'
```

Dry-run the input with `POST /uts/v1/tools/{category}/{tool}/validate` (same body,
no side effects). Discover tools with `GET /uts/v1/tools` and
`GET /uts/v1/tools/catalog`.

**Subscribe to provider events** (delivered as signed HTTP POSTs to a URL you
host):

```bash
curl -s -X POST "$BASE/v2/webhooks/v1/subscriptions/" \
 -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
 -d '{"service_id":"github","connection_id":"<connection_id>",
 "event_types":["push"],"url":"https://myapp.com/hook","name":"deploy"}'
```

Each delivery is HMAC-signed (`X-Webhook-Signature: t=<ts>,v1=<hex>` where `v1` is
`HMAC-SHA256(raw_body, signing_secret)` — sign the **raw body only**). Verify on
receipt, return 2xx fast, dedupe on `X-Webhook-Id`. Verifier snippets and the
deliveries/events endpoints are in
[references/endpoints.md](references/endpoints.md).

**Live event stream (no public URL needed):**

```
wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=<jwt-or-apikey>
```

Auth is on the socket via the `token` query param (a bare JWT or an
`ab0t_sk_…` key). Missing token → close **4001**; bad token/scope → **4003**.
Frames are `{type, data}` JSON. See
[references/endpoints.md](references/endpoints.md#websockets).

**Health & the contract (no auth):**

```bash
curl -s "$BASE/health" # liveness + dependencies
curl -s "$BASE/openapi.json" # THE authoritative contract
curl -s "$BASE/llm.txt" # LLM-oriented service summary
```

## Quick decisions

- **Just calling a provider's native endpoint?** → proxy (`/proxy/{connection_id}/<path>`).
- **Want validated input + structured errors?** → UTS tool execute.
- **Need to receive provider events durably?** → webhook subscription + a hosted receiver.
- **Need events live in a UI/agent, no inbound URL?** → the websocket stream.
- **Unsure of an exact field?** → read `GET {base_url}/openapi.json`.
