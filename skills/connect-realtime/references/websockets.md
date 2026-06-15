# Websockets — the realtime event stream

The live delivery mode. You dial out to the mesh over `wss://`, authenticate on the socket, and events stream in as JSON frames. **No public URL** — ideal for browsers, agents, and anything behind NAT.

Use this for live UIs and agent loops. For durable, retryable server-to-server delivery, use [webhooks.md](webhooks.md) instead. The two share subscriptions: the same subscription that powers webhook POSTs also feeds this stream.

## Contents
- [Endpoint + auth](#endpoint--auth)
- [Close codes](#close-codes)
- [Message protocol](#message-protocol)
- [Filtering the stream](#filtering-the-stream)
- [Keepalive + reconnect](#keepalive--reconnect)
- [Caveat: in-process vs lambda delivery](#caveat-in-process-vs-lambda-delivery)

## Endpoint + auth
```
wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=<jwt_or_api_key>
```

Auth is a **single `token` query parameter**:
- A **mesh JWT** — pass the bare JWT; the socket treats it as `Bearer <token>`. (Passing `Bearer <jwt>` works too.)
- A **mesh API key** — pass the `ab0t_sk_..` value directly; the socket detects the `ab0t_sk_` prefix and treats it as an API key.

A token is **required**. The stream is **scoped to you** server-side — you only ever receive events your token is authorized for, enforced on every frame. Cross-tenant access requires the appropriate mesh permission and is denied otherwise.

CLI shortcuts (no raw socket needed):
```bash
connect tail # connect + print the live stream of events + delivery status
connect listen # subscribe + stream
```

Raw, for a smoke test:
```bash
wscat -c "wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=$CONNECT_API_KEY"
```

## Close codes
| Code | Meaning | What to do |
|------|---------|------------|
| `1000` | normal close | reconnect if you still want the stream |
| `4001` | **auth required** — no token supplied | add `?token=…` |
| `4003` | **auth failed** — bad/expired token, or out-of-scope tenant | refresh the token; check tenant scope |
| `4500` | internal error | back off and reconnect |

If you connect with a JWT, refresh it before expiry — an expired token closes the socket with `4003`; reconnect with a fresh one.

## Message protocol
Frames are JSON objects of the shape `{"type": .., "data": {..}}`.

**Client → server:**
```json
{"type": "ping"}
{"type": "subscribe", "data": {"subscription_ids": ["<sub_id>", ".."]}}
```

**Server → client:**
```json
{"type": "pong"}
{"type": "event.received", "data": {..}}
{"type": "delivery.created", "data": {..}}
{"type": "delivery.updated", "data": {..}}
{"type": "subscription.updated","data": {..}}
{"type": "error", "data": {"code": "..", "message": ".."}}
```

- `event.received` — a provider event arrived for you.
- `delivery.created` / `delivery.updated` — outbound webhook delivery state changed (useful for a "delivery status" UI even if you primarily use webhooks).
- `subscription.updated` — one of your subscriptions changed.
- `error` — a malformed client message or bad subscribe filter; the socket stays open.

### Browser / JS
```js
const ws = new WebSocket(
 `wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=${token}`
);
ws.onopen = () => {
 // optional: narrow to specific subscriptions
 ws.send(JSON.stringify({ type: "subscribe", data: { subscription_ids: subIds } }));
};
ws.onmessage = (e) => {
 const msg = JSON.parse(e.data);
 switch (msg.type) {
 case "event.received": onEvent(msg.data); break;
 case "delivery.updated": onDelivery(msg.data); break;
 case "subscription.updated":onSub(msg.data); break;
 case "pong": /* keepalive ack */ break;
 case "error": console.warn("ws error", msg.data); break;
 }
};
ws.onclose = (e) => {
 if (e.code === 4001 || e.code === 4003) return; // fix auth, don't hot-loop
 scheduleReconnect();
};
```

## Filtering the stream
`subscribe` is **optional**. Send no `subscribe` frame and you receive everything in your scope. Send one to narrow the live stream to specific subscriptions:
```json
{"type": "subscribe", "data": {"subscription_ids": ["<sub_id_1>", "<sub_id_2>"]}}
```
- The server filters the IDs to those you're allowed to access and silently prunes the rest — only your own subscriptions take effect.
- The filter applies to the live stream only; it does not change which webhooks are delivered. Manage what's subscribed at all via the subscriptions API (see the main SKILL.md).

## Keepalive + reconnect
- Send `{"type":"ping"}` on an interval (e.g. every 30s) and expect `{"type":"pong"}` to keep the connection warm through idle proxies.
- The websocket is **best-effort while connected** — there is **no retry or replay** of frames you miss during a disconnect. On reconnect, backfill any gap with `connect webhooks events list` / `GET /v2/webhooks/v1/events`, or rely on webhooks as the durable record.
- Reconnect with exponential backoff on `1000` / `4500`; do **not** hot-loop on `4001` / `4003` (fix the token first).

## Caveat: in-process vs lambda delivery
The realtime stream is fed by the service's **in-process** delivery path. In **lambda-delivery mode** (some prod configurations), the lambda does not publish to the stream, so the socket connects but may show **no live delivery updates**. Confirm with `connect webhooks deliveries list` / `GET /v2/webhooks/v1/deliveries/`. (Tracked as a known infra gap.) For guaranteed delivery regardless of mode, use webhooks.
