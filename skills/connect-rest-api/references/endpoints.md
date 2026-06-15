# Endpoint reference — per-group curl recipes

Raw-HTTP recipes for the ab0t Connect mesh public REST API. The live
`GET {base_url}/openapi.json` is the authoritative contract — when a field here
and the spec disagree, the spec wins.

```
base_url = https://connect.service.ab0t.com (→ connect.service.ab0t.com)
auth = Authorization: Bearer <jwt-or-apikey> (X-API-Key: <apikey> also accepted)
```

Discovery / health endpoints need **no** auth; acting endpoints do.

## Contents

- [Discovery](#discovery)
- [Connections](#connections)
- [Proxy](#proxy)
- [UTS tools](#uts-tools)
- [Webhooks v2 — subscriptions](#webhooks-v2--subscriptions)
- [Webhooks v2 — the public receiver](#webhooks-v2--the-public-receiver)
- [Webhooks v2 — deliveries & events](#webhooks-v2--deliveries--events)
- [Websockets](#websockets)
- [Health & meta](#health--meta)
- [Signature verification (Python / JS / Go)](#signature-verification)
- [Status codes](#status-codes)

Set up once:

```bash
BASE=https://connect.service.ab0t.com
KEY=ab0t_sk_live_..
AUTH=(-H "Authorization: Bearer $KEY") # bash array of header args
```

---

## Discovery

Browse the connectable provider catalog, their tools, OpenAPI specs, and
subscribable events. **No auth required.**

```bash
curl -s "$BASE/services/" # provider catalog
curl -s "$BASE/services/{service_id}" # one provider's metadata
curl -s "$BASE/services/{service_id}/llm.txt" # LLM summary for one provider

curl -s "$BASE/discovery/services" # discovery manifest variant
curl -s "$BASE/discovery/services/{service_id}" # one service from discovery
curl -s "$BASE/discovery/services/{service_id}/events" # subscribable event types
curl -s "$BASE/discovery/manifest" # full manifest
curl -s "$BASE/discovery/categories" # category list
curl -s "$BASE/discovery/capabilities" # capability list
curl -s "$BASE/discovery/search?q=calendar" # search operations across providers
curl -s "$BASE/discovery/openapi/{service_id}" # a provider's OpenAPI spec
curl -s "$BASE/discovery/openapi/{service_id}/operations" # flattened operation list
```

Use these to learn which `service_id` values, scopes, and event types exist
before you create connections or subscriptions.

---

## Connections

A connection is the durable, org-owned, encrypted credential you act through.
`connection_id` is opaque. **Auth required.**

```bash
# list your connections (paginated)
curl -s "$BASE/connections/" "${AUTH[@]}"

# get one
curl -s "$BASE/connections/{connection_id}" "${AUTH[@]}"

# delete one
curl -s -X DELETE "$BASE/connections/{connection_id}" "${AUTH[@]}"
```

**Create — API-key provider** (`POST /connections/{service_id}/api-key`):

```bash
curl -s -X POST "$BASE/connections/{service_id}/api-key" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"api_key":"<provider-secret>","name":"prod"}'
# optional fields: scopes[], additional_config{}, tenant_id, tenant_app_id
# -> { "connection_id": "..", "status": "active", .. }
```

Reserved keys (`access_token`, `refresh_token`, …) inside the body are stripped
server-side — you cannot smuggle OAuth tokens through the api-key endpoint.

**Create — OAuth provider** (two steps; one-time human consent):

```bash
# 1) initiate — returns an authorization_url + state (state expires ~300s)
curl -s -X POST "$BASE/connections/{service_id}/oauth/authorize" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"redirect_uri":"https://app.example.com/callback","scopes":["repo"]}'
# -> { "authorization_url": "..", "state": "..", "expires_in": 300 }
# A human opens authorization_url and approves the requested scopes.

# 2) complete — exchange the code the provider redirected back with
curl -s -X POST "$BASE/connections/{service_id}/oauth/callback" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"code":"<code>","state":"<state>","redirect_uri":"https://app.example.com/callback"}'
# -> { "connection_id": "..", "service_id": "..", "status": "active", "created_at": ".." }
```

`GET /connections/{service_id}/oauth/callback` exists as the browser landing page
for the redirect; the JSON exchange is the `POST`. In an agent loop, emit the
`authorization_url`, let a human approve, then resume on the returned
`connection_id`. Don't re-consent if a valid connection already exists — check
`GET /connections/` first.

---

## Proxy

Transparent passthrough to the provider's **own** native API. You write the
provider's path; the mesh injects the stored credential, refreshes OAuth on
demand, rate-limits, and audits. The HTTP method on your call is the method used
upstream. **Auth required.**

```bash
# GET passthrough (note the provider's native path after the connection_id)
curl -s "$BASE/proxy/{connection_id}/user/repos?per_page=100" "${AUTH[@]}"

# POST with a body
curl -s -X POST "$BASE/proxy/{connection_id}/some/provider/path" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"field":"value"}'

# PUT / PATCH / DELETE all pass through the same way
curl -s -X DELETE "$BASE/proxy/{connection_id}/some/resource/123" "${AUTH[@]}"

# connection metadata WITHOUT making the upstream call
curl -s "$BASE/proxy/{connection_id}/info" "${AUTH[@]}"
```

The proxy strips hop-by-hop and inbound auth headers and replaces the auth header
with the stored credential. Response headers may include `X-Integration-Service`,
`X-Integration-Rate-Limit`, `X-Integration-Cost`. A `401` on a proxy call usually
means the connection's stored upstream credential expired — re-do OAuth or update
the api-key connection.

---

## UTS tools

UTS (Unified Tool Schema) exposes each provider's operations as named, input-
validated tools. Prefer a tool over raw proxy when one exists — richer validation
and structured errors. Discovery is unauthenticated; execute/validate/batch
require auth.

```bash
# discover
curl -s "$BASE/uts/v1/tools" # list tools
curl -s "$BASE/uts/v1/tools/catalog" # grouped catalog
curl -s "$BASE/uts/v1/tools/search?q=calendar" # search
curl -s "$BASE/uts/v1/tools/{category}" # tools in a category
curl -s "$BASE/uts/v1/tools/{category}/{tool}" # one tool's schema
curl -s "$BASE/uts/v1/tools/{category}/{tool}/examples" # usage examples
curl -s "$BASE/uts/v1/connections/{connection_id}/tools" "${AUTH[@]}" # tools for a connection

# validate input WITHOUT executing (same body as execute)
curl -s -X POST "$BASE/uts/v1/tools/{category}/{tool}/validate" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"connection_id":"<connection_id>","input":{ .. }}'

# execute
curl -s -X POST "$BASE/uts/v1/tools/{category}/{tool}/execute" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"connection_id":"<connection_id>",
 "input":{"path_params":{..},"query_params":{..},"body":{..}},
 "options":{}}'

# batch (up to 10 requests; optional parallel)
curl -s -X POST "$BASE/uts/v1/tools/batch" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"parallel":false,
 "requests":[{"id":"r1","method":"GET","path":"/..","connection_id":"<connection_id>"}]}'
```

`input` is the `path_params` / `query_params` / `body` envelope; the tool's schema
(from `GET ../{category}/{tool}`) tells you which keys it needs. A `422` on
execute means input failed schema validation — run `/validate` first.

There is also a connection-scoped proxy-batch at
`POST /batch/{connection_id}` and a generic `POST /batch/` with
`GET /batch/{batch_id}/status` for async batch processing — see the spec.

---

## Webhooks v2 — subscriptions

A subscription says "deliver these event types from this connection." It powers
webhook POSTs and feeds the websocket stream. **Auth required.**

```bash
POST /v2/webhooks/v1/subscriptions/ # create
GET /v2/webhooks/v1/subscriptions/ # list (filter by status/service/connection)
GET /v2/webhooks/v1/subscriptions/{subscription_id} # get one
PUT /v2/webhooks/v1/subscriptions/{subscription_id} # update
DELETE /v2/webhooks/v1/subscriptions/{subscription_id} # delete
POST /v2/webhooks/v1/subscriptions/{subscription_id}/pause
POST /v2/webhooks/v1/subscriptions/{subscription_id}/resume
POST /v2/webhooks/v1/subscriptions/{subscription_id}/test # body: {"event_type":"<type>"}
```

```bash
curl -s -X POST "$BASE/v2/webhooks/v1/subscriptions/" "${AUTH[@]}" \
 -H 'Content-Type: application/json' \
 -d '{"service_id":"github","connection_id":"<connection_id>",
 "event_types":["push"],"url":"https://myapp.com/hook","name":"deploy"}'
# -> the subscription; the create response carries the receiver URL + a SIGNING
# SECRET shown ONCE — store it; you need it to verify deliveries.
```

Create can return **422 missing_scopes** if the chosen `event_types` need OAuth
scopes the connection was never granted — reconnect with the missing scopes, then
retry.

---

## Webhooks v2 — the public receiver

These are the endpoints the **provider** (or a tenant) POSTs inbound events to —
the public ingress side of the mesh. You generally paste the receiver URL into the
provider's dashboard rather than calling it yourself; it is unauthenticated by
design (the mesh verifies provider signatures).

```
POST /v2/webhooks/v1/receiver/{service_id}/{webhook_type}
POST /v2/webhooks/v1/receiver/{service_id}/{connection_id}/{webhook_type}
GET /v2/webhooks/v1/receiver/{service_id}/event_callback # provider handshake (e.g. challenge echo)
POST /v2/webhooks/v1/receiver/{service_id}/event_callback
GET /v2/webhooks/v1/receiver/health
```

The `GET ../event_callback` route handles provider URL-verification handshakes
(e.g. a challenge echo) during subscription setup.

---

## Webhooks v2 — deliveries & events

**Deliveries** are the outbound POSTs the mesh made to your URL. **Events** are
the inbound payloads providers sent you (the arrival audit). Both are scoped to
your org. **Auth required.**

```bash
GET /v2/webhooks/v1/deliveries/ # list (filter by subscription/status/date)
GET /v2/webhooks/v1/deliveries/{delivery_id} # get one
GET /v2/webhooks/v1/deliveries/{delivery_id}/attempts # per-attempt status
POST /v2/webhooks/v1/deliveries/{delivery_id}/retry # retry one failed/timed-out delivery
POST /v2/webhooks/v1/deliveries/bulk-retry # ?subscription_id=&hours=

GET /v2/webhooks/v1/events # inbound events you received
GET /v2/webhooks/v1/events/{event_id} # one inbound event
GET /v2/webhooks/v1/events/stats/summary # counts by event type
```

```bash
curl -s "$BASE/v2/webhooks/v1/deliveries/?status=failed" "${AUTH[@]}"
curl -s -X POST "$BASE/v2/webhooks/v1/deliveries/{delivery_id}/retry" "${AUTH[@]}"
```

Only failed / timed-out deliveries are retryable; retrying a delivered one
returns 400. Delivery IDs are `del_<hex>`; subscription/event IDs are UUIDs. After
a websocket reconnect, backfill any gap from `GET /v2/webhooks/v1/events`.

---

## Websockets

The live realtime stream — dial out over `wss://`, authenticate on the socket, and
events arrive as JSON frames. No public URL required.

```
wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=<jwt-or-apikey>
```

- `token` is **required** and is either a bare mesh JWT or an `ab0t_sk_…` API key;
 the server detects which from the value.
- Close codes: **4001** no token, **4003** bad token / out-of-scope tenant,
 **1000** normal. Reconnect with backoff.

```bash
# smoke test with wscat
wscat -c "wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=$KEY"
```

```js
const ws = new WebSocket(
 `wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=${token}`
);
ws.onmessage = (e) => {
 const msg = JSON.parse(e.data); // {type, data}
 if (msg.type === "event.received") handleEvent(msg.data);
 if (msg.type === "delivery.updated") updateRow(msg.data);
};
```

You send (text JSON): `{"type":"ping"}` and an optional filter
`{"type":"subscribe","data":{"subscription_ids":["<sub_id>"]}}`. The server sends
`{"type":"pong"}`, `{"type":"event.received","data":{..}}`,
`{"type":"delivery.created|updated","data":{..}}`,
`{"type":"subscription.updated","data":{..}}`, and
`{"type":"error","data":{"code":"..","message":".."}}`. The stream is scoped to
your token server-side; send no `subscribe` to get everything in scope. The socket
is **best-effort while connected** — no retry/replay of frames missed during a
disconnect; webhooks are the durable record.

---

## Health & meta

No auth.

```bash
curl -s "$BASE/health" # liveness + dependency status
curl -s "$BASE/health/webhook-runtime" # webhook runtime health
curl -s "$BASE/status" # service config + uptime
curl -s "$BASE/openapi.json" # THE authoritative contract
curl -s "$BASE/llm.txt" # LLM-oriented service summary
curl -s "$BASE/uts/v1/llm.txt" # LLM summary for the UTS surface
```

---

## Signature verification

Each webhook delivery carries:

```
X-Webhook-Signature: t=<unix_ts>,v1=<HMAC-SHA256(raw_body, signing_secret)>
X-Webhook-Id: <delivery id>
X-Webhook-Timestamp: <unix_ts>
```

`v1` is HMAC-SHA256 of the **raw request body bytes only** (lowercase hex) — `t`
is not part of the signed payload. Verify by recomputing and constant-time
comparing. Optionally reject when `|now - t|` exceeds a few minutes (replay
guard). Dedupe on `X-Webhook-Id`.

**Python**

```python
import hmac, hashlib, time

def verify(raw_body: bytes, sig_header: str, secret: str, max_skew=300) -> bool:
 parts = dict(p.split("=", 1) for p in sig_header.split(",") if "=" in p)
 t, v1 = parts.get("t"), parts.get("v1")
 if not t or not v1:
 return False
 if abs(time.time() - int(t)) > max_skew:
 return False
 expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
 return hmac.compare_digest(expected, v1)
```

**JS (Node)**

```js
const crypto = require("crypto");
function verify(rawBody, sigHeader, secret, maxSkew = 300) {
 const parts = Object.fromEntries(
 sigHeader.split(",").map((p) => p.split("=")).filter((kv) => kv.length === 2)
 );
 if (!parts.t || !parts.v1) return false;
 if (Math.abs(Date.now() / 1000 - Number(parts.t)) > maxSkew) return false;
 const expected = crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
 const a = Buffer.from(expected), b = Buffer.from(parts.v1);
 return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

**Go**

```go
import (
 "crypto/hmac"; "crypto/sha256"; "encoding/hex"
 "strconv"; "strings"; "time"
)

func verify(rawBody []byte, sigHeader, secret string, maxSkew int64) bool {
 var t, v1 string
 for _, p := range strings.Split(sigHeader, ",") {
 kv := strings.SplitN(p, "=", 2)
 if len(kv) != 2 { continue }
 switch kv[0] {
 case "t": t = kv[1]
 case "v1": v1 = kv[1]
 }
 }
 if t == "" || v1 == "" { return false }
 ts, err := strconv.ParseInt(t, 10, 64)
 if err != nil { return false }
 if d := time.Now().Unix() - ts; d > maxSkew || d < -maxSkew { return false }
 mac := hmac.New(sha256.New, []byte(secret))
 mac.Write(rawBody)
 expected := hex.EncodeToString(mac.Sum(nil))
 return hmac.Equal([]byte(expected), []byte(v1))
}
```

Always verify against the **raw bytes** of the request body, before any JSON
parsing or re-serialization.

---

## Status codes

| Code | Meaning | Fix |
| --- | --- | --- |
| `200`/`201` | OK | — |
| `400` | Bad request — e.g. wrong connection type for the operation | use a connection of the right service |
| `401` | Missing/expired/wrong-audience token; or a proxy call where the upstream credential expired | refresh the token; re-do OAuth or update the api-key connection |
| `403` | Tenant boundary — that `connection_id` belongs to another org | use one of your own (`GET /connections/`) |
| `404` | Connection or tool not found for your org/version | check the id; `GET /uts/v1/tools` for exact `category/tool` |
| `422` | Input failed schema validation (tool execute); or `missing_scopes` on subscribe | run `/validate`; reconnect with the needed scopes |
| `429` | Provider rate limit (mediated by the mesh) | back off; check `X-Integration-Rate-Limit` |

The live `GET {base_url}/openapi.json` is always the final word on shapes and
parameters.
