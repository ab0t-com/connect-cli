---
name: connect-tools
description: Discover and run ab0t connect mesh UTS (Unified Tool Schema) tools — normalized, input-validated, contract-checked operations over connected providers via /uts/v1/tools/.. Use when you want to run a tool (not a raw proxy call), discover tools (tool catalog, search), validate or execute a tool, batch many tools, or get normalized provider operations with typed input/output schemas and structured errors. Triggers — "run a tool", "execute a tool", "UTS", "unified tool schema", "/uts/v1/tools", "discover tools", "tool catalog", "validate a tool", "batch tools", "normalized provider operations", "typed/safe operations over Slack/GitHub/Google/Xero/Meta Ads/Google Ads". Prefer a UTS tool when one exists; fall back to raw /proxy for any un-wrapped endpoint. CLI is `connect`; host `connect.service.ab0t.com`. Covers discovery, the input envelope, validate-before-execute, batch, structured errors, and curl equivalents.
---

# connect-tools — run validated UTS tools

UTS (Unified Tool Schema) is a **normalized `category/tool/execute` layer** over every
connected provider in the ab0t connect mesh. Instead of crafting a raw provider API
call through the proxy, you call a **named tool** with a **typed input envelope** —
and get input validation, contract pre/postconditions, and **structured errors**
back. This is the surface to prefer for typed, safe, agent-driven operations.

> CLI is `connect`. Default host is
> `connect.service.ab0t.com` (dev: `connect.dev.ab0t.com`). Set `$URL` to the host and `$KEY` to an
> `ab0t_sk_live_…` mesh key for the curl examples.

## What a tool is

Each tool ships a typed contract so callers know exactly what to send and what
they get back:

- **`input.schema`** — JSON Schema for the input envelope (which `path_params` /
 `query_params` / `body` / `headers` it accepts, with patterns and limits).
- **`output.schema`** — JSON Schema for the response (status + body shape).
- **`contract`** — preconditions, postconditions, invariants, and declared
 side-effects (what it reads/writes upstream).
- **`errors` catalog** — every error the tool can return, each with `status`,
 `code`, `category`, `retryable`, and a resolution hint.

So `execute` is **validated before it runs** and returns **structured errors**
(e.g. `{status:403, code:"NOT_IN_CHANNEL", retryable:false, ..}`) — not the raw,
inconsistent provider error you'd get from a bare proxy call.

**Tools call THROUGH the proxy.** Provider headers (Xero `xero-tenant-id`, Google
Ads `developer-token`/`login-customer-id`, Meta `appsecret_proof`, the bearer
token) are injected server-side. You never send credentials in the input.

**No hard gaps.** First-class tools are the typed, discoverable, contract-checked
subset. Any endpoint not yet wrapped as a tool is still reachable via the raw
passthrough `/<proxy>/{connection_id}/<path>` (`GET|POST|PUT|DELETE`). Prefer a
tool when one exists; fall back to `/proxy` otherwise.

## Coverage at a glance

Broad first-class coverage across providers — e.g. Slack, GitHub, Google,
**Xero (33 tools)**, **Meta Ads (23)**, **Google Ads (11)**, plus more. The
catalog is the source of truth; discover, don't guess.

## Discover tools

```bash
connect tools catalog # all tools, grouped by category
connect tools search calendar # search the catalog by keyword
connect tools show slack/post-message --examples # full schema + examples for one tool
connect tools available <connection_id> # tools usable for a connection (alias: tools for)
```

Curl equivalents:

```bash
curl -s "$URL/uts/v1/tools" -H "X-API-Key: $KEY" # list
curl -s "$URL/uts/v1/tools/catalog" -H "X-API-Key: $KEY" # catalog (grouped)
curl -s "$URL/uts/v1/tools/search?q=calendar" -H "X-API-Key: $KEY"
curl -s "$URL/uts/v1/tools/slack/post-message" -H "X-API-Key: $KEY" # schema + examples
curl -s "$URL/uts/v1/connections/<connection_id>/tools" -H "X-API-Key: $KEY"
```

LLM-readable index, the best first stop for an agent: `GET /uts/llm.txt` (also
`GET /uts/v1/llm.txt`).

## The input envelope

`input` is a **structured object**, NOT a flat blob. Use the sub-keys the tool's
schema needs:

- **`path_params`** — values that fill `{placeholders}` in the upstream path
 (e.g. `calendarId`, `act_{id}`). `null` when the path has none.
- **`query_params`** — URL query-string values.
- **`body`** — request body for write tools.
- **`headers`** — extra headers (rare; auth is injected server-side).

Always read the tool's schema first — it tells you exactly which sub-keys and
fields are required (`connect tools show <category>/<tool> --examples`).

## Validate before executing

Cheaper than a failed execute and ideal in an agent loop before a side-effecting
call — it checks `input` against the tool's `input.schema` without running it:

```bash
connect tools validate slack/post-message \
 --data '{"body":{"channel":"C123","text":"hi"}}'
```

```bash
curl -X POST "$URL/uts/v1/tools/slack/post-message/validate" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"input":{"body":{"channel":"C123","text":"hi"}}}'
```

## Execute a tool

`execute` takes a **`connection_id`** (which account to act through) plus the
**`input`** envelope. Ownership and org boundary are enforced server-side — you
only ever execute against your own connections.

```bash
# CLI
connect tools exec slack/post-message --connection <connection_id> \
 --data '{"body":{"channel":"C123","text":"Deploy finished ✅"}}'
```

```bash
# Raw API — URL carries category/tool; body carries connection_id + input
curl -X POST "$URL/uts/v1/tools/slack/post-message/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"conn_..",
 "input":{"body":{"channel":"C123","text":"Deploy finished ✅"}}}'
```

A path-param example (Google Calendar create event):

```bash
curl -X POST "$URL/uts/v1/tools/google-calendar/create_event/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"conn_..",
 "input":{"path_params":{"calendarId":"primary"},
 "body":{"summary":"Standup",
 "start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}}}'
```

## Batch — run many tools in one call

```bash
curl -X POST "$URL/uts/v1/tools/batch" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"requests":[
 {"category":"slack","tool":"post-message",
 "connection_id":"conn_a","input":{"body":{"channel":"C1","text":"one"}}},
 {"category":"slack","tool":"post-message",
 "connection_id":"conn_a","input":{"body":{"channel":"C2","text":"two"}}}
 ]}'
```

Each entry carries its own `connection_id` + `input`; results come back per item.

## Structured errors

Failures are typed, not opaque. Status codes on the `execute` call:

| Code | Meaning | Fix |
| --- | --- | --- |
| `400` | Input failed schema validation, OR connection type doesn't match the tool's category | Run `validate` first; use a connection of the right service |
| `403` | Tenant boundary — that `connection_id` belongs to another org | `connect connections list` to find your own |
| `404` | Unknown tool, or missing connection | `connect tools catalog`; check exact `category/tool` |
| `422` | Missing/ill-formed `connection_id` or `input` in the request body | Check the envelope shape |

The tool's own `errors` catalog defines the provider-level codes too — each with
`status`, `code`, `category`, `retryable`, and `retry_after`/resolution where
applicable (e.g. `429 RATE_LIMITED retryable:true retry_after:60`). Read it via
`connect tools show <category>/<tool>` so you can branch on `code` and honor
`retryable`/`retry_after` instead of parsing free-text errors.

## Agentic loop

```python
# 1. discover (or just read /uts/llm.txt)
tools = GET(f"{URL}/uts/v1/tools/catalog")
# 2. reuse an existing connection — never re-consent if one is active
conn = next(c for c in GET(f"{URL}/connections/")
 if c["service_id"] == "slack" and c["status"] == "active")
# 3. (optional) validate, then execute — headless, repeatable
POST(f"{URL}/uts/v1/tools/slack/post-message/validate",
 {"input": {"body": {"channel": "C123", "text": "hi"}}})
POST(f"{URL}/uts/v1/tools/slack/post-message/execute",
 {"connection_id": conn["connection_id"],
 "input": {"body": {"channel": "C123", "text": "hi"}}})
```

Creating the connection (consent) is the only step that may need a human, and only
for OAuth providers. Everything after — discover, validate, execute, batch — is
normalized and headless.

## Reference

- [references/uts-tools.md](references/uts-tools.md) — endpoint map, the full
 input envelope, the contract/errors anatomy with a worked Slack example, batch,
 and the curl/CLI equivalence table.
