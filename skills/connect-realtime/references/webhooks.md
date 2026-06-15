# Webhooks — HTTP push: subscribe, verify, retry

The HTTP-push delivery mode. Provider event → mesh receiver → matched to your subscriptions → **signed HTTP POST to your URL** → retried on failure → observable via deliveries.

Use this mode for durable server-to-server delivery. For live UI/agent streaming with no inbound URL, use [websockets.md](websockets.md) instead.

## Contents
- [Subscribe](#subscribe)
- [Verify the signature (DO NOT SKIP)](#verify-the-signature-do-not-skip)
- [Build a receiver endpoint](#build-a-receiver-endpoint)
- [Filtering events](#filtering-events)
- [Observe + debug deliveries](#observe--debug-deliveries)

## Subscribe
```bash
connect subscribe github push --connection <conn_id> --url https://myapp.com/hook --name deploy
# prints: the receiver URL to paste into the provider, + a SIGNING SECRET (shown ONCE — store it)
```
Equivalent API: `POST /v2/webhooks/v1/subscriptions/` with `{service_id, connection_id, event_types:[..], url, name}`.
- Receiver URL shape: `https://connect.service.ab0t.com/v2/webhooks/v1/receiver/{service_id}/{connection_id}/{webhook_type}` — paste into the provider's webhook settings.
- The signing secret is returned once on create. Retrieve later: `connect webhooks subscriptions show <sub_id> --secret` (also cached locally per profile).
- IDs: subscription/event IDs are **UUIDs**; delivery IDs are **`del_<hex>`**.
- Create can 422 `missing_scopes` if your chosen `event_types` need OAuth scopes the connection lacks — reconnect with the missing scopes, then retry.

## Verify the signature (DO NOT SKIP)
Every delivery the mesh POSTs to your endpoint carries:
```
X-Webhook-Signature: t=<unix_ts>,v1=<hex>
X-Webhook-Id: <delivery id>
X-Webhook-Timestamp: <unix_ts>
```
**The `v1` HMAC is computed over the RAW REQUEST BODY ONLY** (default `hmac_sha256` delivery method):
`v1 = HMAC_SHA256(key=signing_secret, msg=raw_body_bytes)` rendered as lowercase hex.

⚠️ This is **not** Stripe-style. The `t=` timestamp is for replay-window checks only and is **NOT** part of the signed content. Signing `"{t}.{body}"` will FAIL. Sign the body, nothing else.

Rules: parse `v1` out of the header, HMAC the **exact bytes you received** (verify before JSON-parsing — re-serializing changes bytes), compare in constant time, and reject if `|now - t|` exceeds your window (e.g. 300s).

### Python
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
 return hmac.compare_digest(expected, v1)
```
FastAPI: read `await request.body()` for the raw bytes; do NOT use the parsed model.

### Node / Express
```js
const crypto = require("crypto");
function verify(rawBody /* Buffer */, sigHeader, secret, maxSkew = 300) {
 const parts = Object.fromEntries(sigHeader.split(",").map(p => p.split("=")));
 if (!parts.t || !parts.v1) return false;
 if (Math.abs(Date.now() / 1000 - Number(parts.t)) > maxSkew) return false;
 const expected = crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
 return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(parts.v1));
}
// app.use(express.raw({type: "*/*"})) — capture the raw body, not express.json()
```

### Go
```go
func verify(rawBody []byte, sigHeader, secret string, maxSkew int64) bool {
 var t, v1 string
 for _, p := range strings.Split(sigHeader, ",") {
 kv := strings.SplitN(p, "=", 2)
 if len(kv) == 2 { if kv[0] == "t" { t = kv[1] }; if kv[0] == "v1" { v1 = kv[1] } }
 }
 ts, err := strconv.ParseInt(t, 10, 64)
 if err != nil || v1 == "" { return false }
 if abs(time.Now().Unix()-ts) > maxSkew { return false }
 m := hmac.New(sha256.New, []byte(secret)); m.Write(rawBody)
 return hmac.Equal([]byte(hex.EncodeToString(m.Sum(nil))), []byte(v1))
}
```

## Build a receiver endpoint
Minimum viable receiver: capture raw body → verify → 2xx fast → process async.
```python
@app.post("/hook")
async def hook(request: Request):
 raw = await request.body()
 if not verify(raw, request.headers.get("X-Webhook-Signature", ""), SIGNING_SECRET):
 return Response(status_code=401)
 enqueue(raw) # ack fast; do work off the request path
 return Response(status_code=200) # any 2xx = delivered; non-2xx triggers retry
```
- Respond **2xx within a few seconds**. Slow handlers cause timeouts → retries → duplicates.
- Deliveries are **at-least-once** — dedupe on `X-Webhook-Id`.
- Failed deliveries (non-2xx, timeout, connection error) retry with exponential backoff.
- The receiver URL is public and per-connection; ownership is derived from the `connection_id` in the path, and the mesh tags each event with your `user_id` / `org_id` automatically.

## Filtering events
Subscriptions can carry filters so you only receive what you want. Filters are **JSONPath expressions** against the event payload (repeatable):
```bash
connect subscribe github push --connection <conn> --url <url> \
 --filter '$.ref contains main' --filter '$.repository.name == my-repo'
```
Test what a subscription would receive:
```bash
connect webhooks subscriptions test <sub_id> # body: {"event_type":"<type>"}
```

## Observe + debug deliveries
```bash
connect webhooks deliveries list # recent deliveries (table)
connect webhooks deliveries list --status failed # filter
connect webhooks deliveries show <del_id> --attempts # per-attempt status/code/error
connect webhooks deliveries retry <del_id> # re-queue one
connect webhooks deliveries retry --all-failed # re-queue all failed
connect webhooks events list # raw inbound events (audit)
```
REST: `GET /v2/webhooks/v1/deliveries/`, `GET /v2/webhooks/v1/deliveries/{id}/attempts`, `POST /v2/webhooks/v1/deliveries/{id}/retry`, `POST /v2/webhooks/v1/deliveries/bulk-retry?subscription_id=&hours=`.
- Only `failed` / `timeout` deliveries are retryable; retrying a delivered one returns 400.
- `--output json` on any list for scripting/agents.
- `deliveries show <id>` is a point-read by `del_<hex>` id.
