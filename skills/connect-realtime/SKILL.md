---
name: connect-realtime
description: Receive events from providers connected through the ab0t Connect mesh, and choose between the two delivery modes — WEBHOOKS (HTTP push to your URL) and WEBSOCKETS (a live realtime event stream). Use when a user wants to "receive events", "subscribe to events", "get webhooks", "set up a webhook receiver", "verify a webhook signature", "HMAC signature verification", "handle deliveries / retries", "listen to a live event stream", "open a websocket", "wss ?token", "realtime events", "stream events to a browser/agent", "events behind NAT with no public URL", or is deciding "push vs stream" / "webhooks vs websockets". Covers subscribe (create/list/pause/resume/test), the public receiver URL, signature verification in Python/JS/Go, deliveries (list/retry/bulk-retry/attempts), the inbound events audit, AND the WebSocket realtime stream (auth via wss ?token=, message protocol, close codes). Serves the integrator building durable server-to-server event pipelines (webhooks) and the builder of live UIs and agents (websockets).
---

# Receiving events — webhooks vs websockets

A provider you connected fires an event. Two ways to receive it through the Connect mesh:

| | **Webhooks** (HTTP push) | **Websockets** (live stream) |
|---|---|---|
| How | Mesh POSTs each delivery to a URL you host | You open a `wss://` socket and events arrive live |
| Needs a public URL? | **Yes** — an inbound HTTPS endpoint | **No** — you dial out (great behind NAT) |
| Durability | At-least-once, retried with backoff, replayable | Best-effort while connected; no retry/replay |
| Verify? | **You verify the HMAC signature** on every POST | Auth is on the socket; no per-message signing |
| Best for | Server-to-server pipelines, durable processing, audit | Live UIs, dashboards, agents, browsers, anything ephemeral |

**Choose webhooks** when you run a server and need every event durably, with retries and a delivery log.
**Choose websockets** when you want events live in a browser, an agent loop, or any client that can't expose an inbound URL. Many apps use **both**: webhooks as the system of record, a websocket to push live updates into the UI.

> The Connect CLI is `connect`; commands below work as-is. Host: `connect.service.ab0t.com`; websocket scheme is `wss://`.

Every event flows **through a `connection_id`** you created first (see the connect-connections skill).

---

## Webhooks — the short version

```bash
connect subscribe github push --connection <conn_id> --url https://myapp.com/hook --name deploy
# prints: the receiver URL to paste into the provider, + a SIGNING SECRET (shown ONCE — store it)
```

You then host an endpoint that:
1. reads the **raw request body**,
2. verifies the `X-Webhook-Signature` HMAC against your signing secret,
3. returns **2xx within a few seconds**, and
4. processes off the request path.

```python
import hmac, hashlib, time

def verify(raw_body: bytes, sig_header: str, secret: str, max_skew=300) -> bool:
 parts = dict(p.split("=", 1) for p in sig_header.split(",") if "=" in p)
 t, v1 = parts.get("t"), parts.get("v1")
 if not t or not v1:
 return False
 if abs(time.time() - int(t)) > max_skew: # replay guard
 return False
 expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
 return hmac.compare_digest(expected, v1) # sign the BODY, nothing else
```

Deliveries are **at-least-once** — dedupe on `X-Webhook-Id`. Non-2xx / timeout → retried with backoff.

**Full HTTP-push reference (subscribe, receiver, signature in Python/JS/Go, filters, retries/dedupe):**
[references/webhooks.md](references/webhooks.md)

---

## Websockets — the realtime stream (the big one)

No public URL. Open one socket, authenticate with a token in the query string, and events stream in live.

### Connect

```
wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=<jwt_or_api_key>
```

- `token` is **either** a mesh JWT (a bare JWT; the mesh treats it as `Bearer <token>`) **or** an `ab0t_sk_..` API key. The socket detects which from the value.
- A token is **required**. Without it the server closes with **4001** (auth required). An invalid token or out-of-scope tenant closes with **4003** (auth failed).
- Quick smoke test from a shell, or the CLI shortcut:

```bash
connect tail # live stream of inbound events + delivery status (top-level shortcut)
connect listen # same family; subscribe + print the live stream

# raw, with wscat:
wscat -c "wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=$CONNECT_API_KEY"
```

### Browser / JS

```js
const ws = new WebSocket(
 `wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=${token}`
);
ws.onmessage = (e) => {
 const msg = JSON.parse(e.data); // {type, data}
 if (msg.type === "event.received") handleEvent(msg.data);
 if (msg.type === "delivery.updated") updateDeliveryRow(msg.data);
};
ws.onclose = (e) => {
 // 4001 = no token, 4003 = bad token/scope, 1000 = normal. Reconnect with backoff.
};
```

### Message protocol

You send (text JSON):

```json
{"type": "ping"}
{"type": "subscribe", "data": {"subscription_ids": ["<sub_id>", ".."]}}
```

The server sends:

```json
{"type": "pong"}
{"type": "event.received", "data": {..}}
{"type": "delivery.created", "data": {..}}
{"type": "delivery.updated", "data": {..}}
{"type": "subscription.updated","data": {..}}
{"type": "error", "data": {"code": "..", "message": ".."}}
```

- You only ever receive events your token is authorized for; the stream is **scoped to you** server-side.
- `subscribe` is an **optional filter** — pass `subscription_ids` to narrow the stream to specific subscriptions (server prunes any you can't access). Send no `subscribe` and you get everything in your scope.
- Send a periodic `{"type":"ping"}` and expect `{"type":"pong"}` to keep the socket warm.

**Full websocket reference (auth detail, close codes, filtering, reconnect, the lambda-mode caveat):**
[references/websockets.md](references/websockets.md)

---

## Manage subscriptions (both modes start here)

A **subscription** says "deliver these event types from this connection." It powers webhook POSTs and feeds the websocket stream.

```bash
connect webhooks subscriptions list # your subscriptions (table)
connect webhooks subscriptions show <sub_id> # detail (--secret to reveal signing secret)
connect webhooks subscriptions test <sub_id> # fire a synthetic event to check wiring
connect webhooks subscriptions pause <sub_id> # stop delivering (keeps config)
connect webhooks subscriptions resume <sub_id> # resume
```

REST equivalents (`{api_url}` = `https://connect.service.ab0t.com`):

```
POST /v2/webhooks/v1/subscriptions/ create {service_id, connection_id, event_types[], url, name}
GET /v2/webhooks/v1/subscriptions/ list (paginated; filter by status/service/connection)
GET /v2/webhooks/v1/subscriptions/{id} get one
PUT /v2/webhooks/v1/subscriptions/{id} update
DELETE /v2/webhooks/v1/subscriptions/{id} delete
POST /v2/webhooks/v1/subscriptions/{id}/test send a test event
POST /v2/webhooks/v1/subscriptions/{id}/pause pause
POST /v2/webhooks/v1/subscriptions/{id}/resume resume
```

```bash
curl -X POST https://connect.service.ab0t.com/v2/webhooks/v1/subscriptions/ \
 -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
 -d '{"service_id":"github","connection_id":"<conn_id>","event_types":["push"],
 "url":"https://myapp.com/hook","name":"deploy"}'
```

> Create can return **422 `missing_scopes`** if the chosen `event_types` need OAuth scopes the connection was never granted — reconnect the connection with the missing scopes, then retry.

---

## Observe deliveries + the inbound events audit

Deliveries are the **outbound POSTs to your webhook URL** — list them, inspect per-attempt status, and retry.

```bash
connect webhooks deliveries list # recent deliveries
connect webhooks deliveries list --status failed
connect webhooks deliveries show <del_id> --attempts # per-attempt code/error
connect webhooks deliveries retry <del_id> # re-queue one
connect webhooks deliveries retry --all-failed # bulk re-queue failed
connect webhooks events list # raw INBOUND events (audit of what arrived)
```

REST:

```
GET /v2/webhooks/v1/deliveries/ list (filter by subscription/status/date)
GET /v2/webhooks/v1/deliveries/{id} get one
POST /v2/webhooks/v1/deliveries/{id}/retry retry one failed/timed-out delivery
GET /v2/webhooks/v1/deliveries/{id}/attempts all attempts for a delivery
POST /v2/webhooks/v1/deliveries/bulk-retry bulk retry (?subscription_id=&hours=)
GET /v2/webhooks/v1/events inbound events you received (your scope only)
GET /v2/webhooks/v1/events/{id} one inbound event
GET /v2/webhooks/v1/events/stats/summary counts by event type
```

- Inbound **events** (what providers sent you) and outbound **deliveries** (what the mesh POSTed to your URL) are distinct: events are the audit of arrivals; deliveries are the fan-out attempts to your receiver.
- Only failed / timed-out deliveries are retryable; retrying a delivered one returns 400.
- `--output json` on any list for scripting and agents.

---

## Pick your path

- **Server-to-server, must not lose events** → webhooks. Subscribe, host a receiver, verify the signature, ack 2xx, lean on retries + the delivery log. → [references/webhooks.md](references/webhooks.md)
- **Live UI / agent / browser / behind NAT** → websockets. Open `wss://…/v2/webhooks/v1/ws?token=…`, read `{type,data}` frames. → [references/websockets.md](references/websockets.md)
- **Both** → subscriptions feed both; use webhooks as the durable record and the socket for live UI.
