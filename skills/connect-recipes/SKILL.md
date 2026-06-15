---
name: connect-recipes
description: End-to-end COOKBOOK of worked, copy-pasteable examples for building real things on the ab0t Connect mesh — ties the surfaces (connections, proxy, tools, realtime, auth) together into complete goal→connect→act→done recipes. Use when a user asks "how do I", "give me an example", "show me a recipe", "an end-to-end example", "a worked example", "a cookbook", or wants to "post a Slack message", "sync invoices" / "sync Xero invoices", "stream events to a UI" / "stream GitHub events to a browser", "build a webhook pipeline" / "durable webhook pipeline", "verify HMAC signatures", "build a multi-tenant SaaS" / "let my users connect their own accounts" / "per-customer connections", "build an agent on the mesh" / "an AI agent that discovers and runs tools", or just "tie it together" / "how do I actually build X". Each recipe gives real connect (connect) commands AND curl. For the deep mechanics of any single surface, this composes the connect-connections, connect-proxy, connect-tools, connect-realtime, and connect-auth skills.
---

# Connect recipes — worked, end-to-end examples

The "show me how to actually build X" cookbook for the ab0t Connect mesh. Each
recipe is a full path: **goal → connect (consent once) → act (headless) → done**,
with real CLI and curl. For the deep mechanics of any surface, the per-surface
skills are the reference; this stitches them into complete builds.

> The CLI is `connect` (ticket 7f3a);
> the commands below work as-is. Default host
> `connect.service.ab0t.com` (dev: `connect.dev.ab0t.com`). In curl, `URL` = that host and
> `KEY` = an `ab0t_sk_live_…` mesh key. The live `GET /openapi.json` is the
> authoritative contract.

## The shape every recipe follows

1. **Auth to the mesh** — `export CONNECT_API_KEY=ab0t_sk_live_…` (headless) or
 `connect login` (human). Orthogonal to connecting providers.
2. **Connect the provider** — once. API-key services self-connect; OAuth
 services need one human consent, then act headless forever.
3. **Act through the `connection_id`** — via proxy, tools, or events.
4. **Done** — the connection persists; repeat the act step with no human.

## Recipe index

| # | Goal | Surfaces | Where |
| --- | --- | --- | --- |
| 1 | Post a Slack message | connect (OAuth) → tools / proxy | inline below |
| 2 | Sync Xero invoices | connect (OAuth) → tools, paging | inline below |
| 3 | Stream GitHub events to a live UI | connect (OAuth) → websocket | inline below |
| 4 | Durable webhook pipeline | connect → subscribe → HMAC verify → retries | inline below |
| 5 | Multi-tenant SaaS — each customer connects their own accounts | per-org/per-user connections, isolation | [references/multi-tenant-saas.md](references/multi-tenant-saas.md) |
| 6 | An AI agent that discovers + runs tools | headless connect → discover → validate → execute → recover | [references/agent-patterns.md](references/agent-patterns.md) |

---

## Recipe 1 — Post a Slack message

**Goal:** send a message to a Slack channel from your app or a script.

**Connect** (Slack is OAuth — one human consent, then headless forever):

```bash
connect add slack # opens a browser once; stores the connection
connect connections list --service slack
# -> note the connection_id (call it $CONN)
```

**Act — preferred: the UTS tool** (validated input, structured errors):

```bash
connect tools exec slack/post-message --connection $CONN \
 --data '{"body":{"channel":"C0123456789","text":"Deploy finished ✅"}}'
```

curl:

```bash
curl -X POST "$URL/uts/v1/tools/slack/post-message/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"'"$CONN"'",
 "input":{"body":{"channel":"C0123456789","text":"Deploy finished ✅"}}}'
```

**Act — fallback: the raw proxy** (Slack's own `chat.postMessage`, for fields no
tool wraps):

```bash
connect call $CONN POST /chat.postMessage \
 --data '{"channel":"C0123456789","text":"Deploy finished ✅"}'
# = POST $URL/proxy/$CONN/chat.postMessage (the Slack token is injected for you)
```

**Done.** The same exec/call runs forever with no human. Wrong-type connection →
`400`; another org's `connection_id` → `403`; bad `input` → `422` (run
`connect tools validate slack/post-message --data '…'` first).

---

## Recipe 2 — Sync Xero invoices

**Goal:** pull all invoices from a connected Xero org, handling the tenant header
and paging.

**Connect** (Xero is OAuth; the `xero-tenant-id` is captured at consent and
injected on every later call — you never send it):

```bash
connect add xero
connect connections list --service xero # -> $CONN
connect connections show $CONN # confirm status=active, see tenant_id
```

**Act — the tool** (Xero has broad first-class tool coverage; discover, don't
guess the name):

```bash
connect tools search invoice # find the Xero invoices tool key
connect tools show xero/list-invoices --examples
connect tools exec xero/list-invoices --connection $CONN \
 --data '{"query_params":{"page":1}}'
```

**Act — the proxy fallback** (Xero's native Accounting API; the tenant header is
injected server-side):

```bash
connect call $CONN GET '/api.xro/2.0/Invoices?page=1'
```

curl, paging until a page returns fewer than the page size:

```bash
page=1
while : ; do
 body=$(curl -s "$URL/proxy/$CONN/api.xro/2.0/Invoices?page=$page" -H "X-API-Key: $KEY")
 n=$(printf '%s' "$body" | jq '.Invoices | length')
 printf '%s' "$body" | jq -c '.Invoices[]' # .. persist each invoice ..
 [ "$n" -lt 100 ] && break # Xero pages by 100; short page = last
 page=$((page+1))
done
```

**Tenant + paging notes:** the mesh injects `xero-tenant-id` for you — it never
appears in your call. The proxy does **not** auto-paginate; honor Xero's
page-by-100 convention (loop `?page=N` until a short page). One Xero org per
connection — connect each org separately and pick the matching `$CONN`.

**Done.** Re-run on a schedule headlessly; OAuth tokens refresh transparently.

---

## Recipe 3 — Stream GitHub events to a live UI

**Goal:** show GitHub events (pushes, PRs) live in a browser or agent — **no
public URL**, you dial out over a websocket.

**Connect + subscribe** (the subscription defines which events feed the stream):

```bash
connect add github
connect connections list --service github # -> $CONN
connect subscribe github push --connection $CONN --name live-ui
connect webhooks subscriptions list # -> note the sub_id
```

**Act — open the websocket.** Auth is a token in the query string (a bare mesh
JWT or an `ab0t_sk_…` key — the socket detects which):

```
wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=<jwt_or_api_key>
```

Browser / JS — events arrive live as `{type, data}` frames:

```js
const ws = new WebSocket(
 `wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=${token}`
);
ws.onopen = () => {
 // optional: narrow the stream to specific subscriptions
 ws.send(JSON.stringify({ type: "subscribe",
 data: { subscription_ids: [SUB_ID] } }));
};
ws.onmessage = (e) => {
 const msg = JSON.parse(e.data); // {type, data}
 if (msg.type === "event.received") renderEvent(msg.data); // a GitHub event, live
};
ws.onclose = (e) => {
 // 4001 = no token, 4003 = bad/expired token or out-of-scope, 1000 = normal.
 // Reconnect with backoff; refresh a JWT before it expires.
};
setInterval(() => ws.readyState === 1 && ws.send('{"type":"ping"}'), 30000);
```

Shell smoke test:

```bash
connect tail # CLI live stream
wscat -c "wss://connect.service.ab0t.com/v2/webhooks/v1/ws?token=$CONNECT_API_KEY"
```

**Done.** The stream is scoped to you server-side — you only ever receive events
your token is authorized for. The socket is best-effort while connected (no
retry/replay) — pair it with Recipe 4 if you must not lose events.

---

## Recipe 4 — Durable webhook pipeline

**Goal:** a server that receives provider events durably, with at-least-once
delivery, HMAC verification, and server-side retries.

**Connect + subscribe** (gives you a receiver URL and a signing secret shown
**once** — store it):

```bash
connect add github # -> $CONN
connect subscribe github push --connection $CONN \
 --url https://myapp.com/hook --name deploy
# prints: the receiver URL (already set as your subscription target)
# + a SIGNING SECRET (shown ONCE — save it now)
```

curl equivalent:

```bash
curl -X POST "$URL/v2/webhooks/v1/subscriptions/" \
 -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
 -d '{"service_id":"github","connection_id":"'"$CONN"'","event_types":["push"],
 "url":"https://myapp.com/hook","name":"deploy"}'
```

**Act — host a receiver** that (1) reads the **raw** body, (2) verifies the HMAC,
(3) returns 2xx fast, (4) processes off the request path. Every delivery carries:

```
X-Webhook-Signature: t=<unix_ts>,v1=<HMAC-SHA256(raw_body, signing_secret)>
X-Webhook-Id: <delivery id> # dedupe on this — delivery is at-least-once
X-Webhook-Timestamp: <unix_ts>
```

`v1` signs the **raw request body only** (lowercase hex); `t` is **not** part of
the signed payload (this is not the Stripe `t.body` scheme):

```python
import hmac, hashlib, time
from flask import Flask, request, abort

app = Flask(__name__)
SECRET = b"<your signing secret>" # the one shown once at subscribe
seen = set() # use a durable store in production

def verify(raw_body: bytes, sig_header: str, secret: bytes, max_skew=300) -> bool:
 parts = dict(p.split("=", 1) for p in sig_header.split(",") if "=" in p)
 t, v1 = parts.get("t"), parts.get("v1")
 if not t or not v1 or abs(time.time() - int(t)) > max_skew: # replay guard
 return False
 expected = hmac.new(secret, raw_body, hashlib.sha256).hexdigest()
 return hmac.compare_digest(expected, v1) # sign the BODY only

@app.post("/hook")
def hook():
 raw = request.get_data() # RAW bytes, pre-parse
 if not verify(raw, request.headers.get("X-Webhook-Signature", ""), SECRET):
 abort(401)
 did = request.headers.get("X-Webhook-Id")
 if did in seen:
 return "", 200 # idempotent on redelivery
 seen.add(did)
 enqueue(raw) # process OFF the request path
 return "", 200 # ack within a few seconds
```

**Server-side retries + delivery log** (the mesh re-POSTs failed/timed-out
deliveries with backoff — you don't build retry yourself):

```bash
connect webhooks deliveries list --status failed
connect webhooks deliveries show <del_id> --attempts # per-attempt code/error
connect webhooks deliveries retry <del_id> # re-queue one
connect webhooks deliveries retry --all-failed # bulk re-queue
connect webhooks events list # audit of inbound arrivals
```

**Done.** Inbound **events** (what providers sent) and outbound **deliveries**
(what the mesh POSTed to you) are distinct surfaces. Non-2xx or timeout → retried
with backoff; only failed/timed-out deliveries are retryable. A `422
missing_scopes` at subscribe means the connection lacks an OAuth scope the event
needs — reconnect with it and retry.

---

## Two bigger recipes (in references/)

- **Recipe 5 — Multi-tenant SaaS** — the commercial pattern: each of your
 customers connects THEIR OWN accounts; per-org/per-user connections; headless
 API-key connections for service accounts; acting through each customer's
 `connection_id`; server-side tenant isolation. →
 [references/multi-tenant-saas.md](references/multi-tenant-saas.md)

- **Recipe 6 — An AI agent that discovers + runs tools** — headless connect →
 discover tools (`/uts/llm.txt`, catalog) → validate → execute → recover from
 structured errors (honoring `retryable` / `retry_after`). →
 [references/agent-patterns.md](references/agent-patterns.md)

## Cross-recipe gotchas

- **Consent once, act forever.** Never re-consent if an `active` connection
 already exists — `connect connections list` and reuse the `connection_id`.
- **Prefer a tool over the proxy** when one exists (validation + structured
 errors); fall back to `/proxy/{connection_id}/<native path>` for anything not
 yet wrapped.
- **Tenancy is server-side.** You only ever see/act on your own org's
 connections — another org's `connection_id` is `403`, always.
- **Headless everywhere.** Set `CONNECT_API_KEY` and add `-o json` for
 agent-parseable output; nothing after consent needs a browser.
