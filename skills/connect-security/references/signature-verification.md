# Verifying inbound webhook signatures (HMAC)

Every webhook the Connect mesh POSTs to your receiver URL is signed. **Verify the signature before you trust the delivery** — it proves the event came from the mesh, was not tampered with, and is not a replay. This is the one security step you implement on your side; everything else (encryption at rest, token rotation, tenant isolation) is enforced server-side.

This is the single source for the verify pattern; the SKILL.md body shows the short Python form and links here.

## Headers on every delivery

```
X-Webhook-Signature: t=<unix_ts>,v1=<hex>
X-Webhook-Id: <delivery id>
X-Webhook-Timestamp: <unix_ts>
```

## The rule

```
v1 = HMAC_SHA256(key=signing_secret, msg=raw_body_bytes) # lowercase hex
```

- The HMAC is computed over the **raw request body ONLY**.
- This is **not** Stripe-style. The `t=` value is for the replay-window check only and is **NOT** part of the signed content. Signing `"{t}.{body}"` will FAIL.
- Verify the **exact bytes you received** — verify *before* JSON-parsing, because re-serializing changes the bytes and breaks the HMAC.
- Compare in **constant time**.
- Reject if `|now - t|` exceeds your window (e.g. 300s) — this is the replay guard.

The signing secret is shown **once** when you subscribe — store it. (Retrieve later via `connect webhooks subscriptions show <sub_id> --secret`.)

## Python

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

## Node / Express

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

## Go

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

## Minimal receiver

```python
@app.post("/hook")
async def hook(request: Request):
 raw = await request.body()
 if not verify(raw, request.headers.get("X-Webhook-Signature", ""), SIGNING_SECRET):
 return Response(status_code=401) # reject unsigned / forged / replayed
 enqueue(raw) # ack fast; process off the request path
 return Response(status_code=200)
```

- Respond **2xx within a few seconds**; slow handlers time out → retries → duplicates.
- Deliveries are **at-least-once** — dedupe on `X-Webhook-Id`.
- A failed verification (401) is the correct response to anything you can't authenticate.

The realtime `wss://` stream has **no per-message signature** — its authenticity comes from the authenticated socket itself (token on connect, scoped to your org). Signature verification applies to HTTP-push webhooks only.
