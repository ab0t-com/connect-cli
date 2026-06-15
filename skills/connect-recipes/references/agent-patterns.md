# Recipe 6 — An AI agent that discovers + runs tools

Build an autonomous agent on the Connect mesh: it authenticates headless,
discovers the tools available, validates before side-effects, executes through a
connection, and recovers from structured errors. The whole loop runs with no
human after the one-time consent.

> CLI `connect`. Host
> `connect.service.ab0t.com`. `URL` = host,
> `KEY` = an `ab0t_sk_live_…` mesh key.

## Contents

- [The core split the agent is built on](#the-core-split-the-agent-is-built-on)
- [Step 0 — headless auth](#step-0--headless-auth)
- [Step 1 — ensure a connection](#step-1--ensure-a-connection-the-only-step-that-may-need-a-human)
- [Step 2 — discover tools](#step-2--discover-tools-dont-guess)
- [Step 3 — the input envelope](#step-3--the-input-envelope)
- [Step 4 — validate before execute](#step-4--validate-before-a-side-effecting-execute)
- [Step 5 — execute through the connection](#step-5--execute-through-the-connection)
- [Step 6 — recover from structured errors](#step-6--recover-from-structured-errors)
- [The full loop, end to end](#the-full-loop-end-to-end)
- [Beyond tools — proxy + events](#beyond-tools--proxy--events-from-an-agent)

## The core split the agent is built on

**Consent (one-time, identity-granting) is separate from action (repeated,
headless).** The connection is the durable consent artifact. Once it exists, the
agent acts through `connection_id` forever — no human in the loop.

## Step 0 — headless auth

The agent authenticates to the mesh with an API key (never a browser). The key
wins over any on-disk credential, so a one-shot run mutates no state:

```bash
CONNECT_API_KEY=ab0t_sk_live_.. connect whoami -o json
```

A `401` means the key is missing, malformed, expired, or its audience doesn't
match the host — rotate it in the mesh, re-export, retry. In code, send
`X-API-Key: $KEY` on every request.

## Step 1 — ensure a connection (the only step that may need a human)

The agent reuses an existing `active` connection and **never re-consents** if one
exists. Only OAuth providers, and only when no connection exists, need a one-time
human approval (emit the URL, a human/HITL approves, the agent resumes):

```python
def ensure_connection(service):
 conns = GET("/connections/") # X-API-Key auth
 for c in conns:
 if c["service_id"] == service and c["status"] == "active":
 return c["connection_id"] # reuse — no re-consent
 if is_api_key_service(service): # Stripe/SendGrid/Twilio/…
 return POST(f"/connections/{service}/api-key",
 {"api_key": SECRET, "name": "agent"})["connection_id"]
 # OAuth: the one human touch
 url = POST(f"/connections/{service}/oauth/authorize",
 {"redirect_uri": CALLBACK})["authorization_url"]
 ask_human(url) # HITL approves once
 return poll_until_connection(service)["connection_id"]
```

## Step 2 — discover tools (don't guess)

The single best first stop for an agent is the LLM-readable index; then the
catalog and per-connection availability:

```bash
curl -s "$URL/uts/llm.txt" # LLM-readable tool index
curl -s "$URL/uts/v1/tools/catalog" -H "X-API-Key: $KEY" # grouped catalog
curl -s "$URL/uts/v1/tools/search?q=invoice" -H "X-API-Key: $KEY" # keyword search
curl -s "$URL/uts/v1/connections/$CONN/tools" -H "X-API-Key: $KEY" # usable for this conn
curl -s "$URL/uts/v1/tools/slack/post-message" -H "X-API-Key: $KEY" # the tool's schema
```

Each tool ships a typed contract: `input.schema`, `output.schema`, a `contract`
(pre/postconditions + declared side-effects), and an `errors` catalog (every
error with `status`, `code`, `category`, `retryable`, resolution hint). Read the
schema before building the `input` envelope.

## Step 3 — the input envelope

`input` is a **structured object**, not a flat blob. Use the sub-keys the tool's
schema declares:

- `path_params` — fill `{placeholders}` in the upstream path.
- `query_params` — URL query values.
- `body` — request body for write tools.
- `headers` — rare; auth is injected server-side, never send credentials.

## Step 4 — validate before a side-effecting execute

Cheaper than a failed execute and ideal in an agent loop. It checks `input`
against `input.schema` without running the tool:

```bash
curl -X POST "$URL/uts/v1/tools/slack/post-message/validate" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"input":{"body":{"channel":"C123","text":"hi"}}}'
```

## Step 5 — execute through the connection

The URL carries `category/tool`; the body carries `connection_id` + `input`.
Ownership and the org boundary are enforced server-side:

```bash
curl -X POST "$URL/uts/v1/tools/slack/post-message/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"'"$CONN"'",
 "input":{"body":{"channel":"C123","text":"Done ✅"}}}'
```

Many calls in one round-trip with batch (each entry carries its own
`connection_id` + `input`; results come back per item):

```bash
curl -X POST "$URL/uts/v1/tools/batch" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"requests":[
 {"category":"slack","tool":"post-message","connection_id":"'"$CONN"'",
 "input":{"body":{"channel":"C1","text":"one"}}},
 {"category":"slack","tool":"post-message","connection_id":"'"$CONN"'",
 "input":{"body":{"channel":"C2","text":"two"}}}]}'
```

## Step 6 — recover from structured errors

Failures are typed, not opaque — branch on `code`, honor `retryable` /
`retry_after`, never parse free text. Request-level codes on `execute`:

| Code | Meaning | Agent action |
| --- | --- | --- |
| `400` | Input failed validation, or connection type ≠ the tool's category | Re-`validate`; pick a connection of the right service |
| `403` | Tenant boundary — that `connection_id` is another org's | List your own connections; never act cross-org |
| `404` | Unknown tool, or missing connection | Re-`catalog`; check exact `category/tool` |
| `422` | Missing/ill-formed `connection_id` or `input` | Fix the envelope shape |

The tool's own `errors` catalog defines provider-level codes too — each with
`status`, `code`, `category`, `retryable`, and `retry_after` where applicable
(e.g. `429 RATE_LIMITED retryable:true retry_after:60`, `403 NOT_IN_CHANNEL
retryable:false`). Read it via `connect tools show <category>/<tool>` and let the
agent decide: retry-after-backoff vs. surface-to-human vs. pick another path.

```python
resp = POST(f"/uts/v1/tools/{cat}/{tool}/execute",
 {"connection_id": conn, "input": envelope})
if resp.ok:
 return resp.json()
err = resp.json().get("error", {}) # {code, retryable, retry_after, ..}
if err.get("retryable"):
 sleep(err.get("retry_after", 5)); return retry()
raise NonRetryable(err["code"]) # e.g. NOT_IN_CHANNEL — escalate
```

## The full loop, end to end

```python
KEY = os.environ["CONNECT_API_KEY"] # headless, no browser
conn = ensure_connection("slack") # reuse-or-consent-once (Step 1)
tools = GET("/uts/v1/tools/catalog") # discover (Step 2)
envelope = {"body": {"channel": "C123", "text": "Deploy finished ✅"}}
POST("/uts/v1/tools/slack/post-message/validate", {"input": envelope}) # Step 4
resp = POST("/uts/v1/tools/slack/post-message/execute",
 {"connection_id": conn, "input": envelope}) # Step 5
handle(resp) # branch on structured error (Step 6)
```

Creating the connection (consent) is the only step that may need a human, and
only for OAuth providers. Everything else — discover, validate, execute, batch,
recover — is normalized and headless. Pair this with `-o json` / `X-API-Key`
throughout so the agent parses structured output, and reuse connections via
`GET /connections/` so it never re-consents.

## Beyond tools — proxy + events from an agent

- **Proxy fallback:** for any provider endpoint no tool wraps, the agent calls
 `{METHOD} $URL/proxy/{connection_id}/<native path>` with `X-API-Key` — the
 credential is injected, the response is the provider's own.
- **Live input:** an agent that reacts to provider events opens the websocket
 (`wss://…/v2/webhooks/v1/ws?token=$KEY`) and reads `{type, data}` frames — no
 public URL needed (see Recipe 3). For durable, must-not-lose event processing,
 back it with a webhook pipeline (Recipe 4).
