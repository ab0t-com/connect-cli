# UTS tools — endpoint map, envelope, contract anatomy

Deep reference for the `/uts/v1/tools/..` surface. `$URL` = the connect mesh host
(`https://connect.service.ab0t.com`);
`$KEY` = an `ab0t_sk_live_…` mesh key. CLI is `connect`.

## Contents
- [Endpoint map](#endpoint-map)
- [CLI ↔ curl equivalence](#cli--curl-equivalence)
- [The input envelope](#the-input-envelope)
- [Tool anatomy: schema + contract + errors](#tool-anatomy-schema--contract--errors)
- [A worked example: slack/post-message](#a-worked-example-slackpost-message)
- [Batch](#batch)
- [Why a tool over a raw proxy call](#why-a-tool-over-a-raw-proxy-call)

## Endpoint map

| Method + path | Purpose |
| --- | --- |
| `GET /uts/v1/tools` | List all tools |
| `GET /uts/v1/tools/catalog` | Catalog, grouped by category |
| `GET /uts/v1/tools/search?q=<keyword>` | Search the catalog |
| `GET /uts/v1/tools/{category}/{tool}` | One tool: input/output schema + examples |
| `POST /uts/v1/tools/{category}/{tool}/validate` | Check `input` against the schema; no execution |
| `POST /uts/v1/tools/{category}/{tool}/execute` | Run the tool (`connection_id` + `input`) |
| `POST /uts/v1/tools/batch` | Run many tool calls in one request |
| `GET /uts/v1/connections/{connection_id}/tools` | Tools usable for a specific connection |
| `GET /uts/llm.txt`, `GET /uts/v1/llm.txt` | LLM-readable index (best first stop for an agent) |

The URL carries `{category}/{tool}`; the request body carries `connection_id` and
`input`. `validate` takes just `{"input": {..}}`.

## CLI ↔ curl equivalence

| `connect` | Raw API |
| --- | --- |
| `tools catalog` | `GET /uts/v1/tools/catalog` |
| `tools search <q>` | `GET /uts/v1/tools/search?q=<q>` |
| `tools show <cat>/<tool> --examples` | `GET /uts/v1/tools/<cat>/<tool>` |
| `tools available <conn>` (alias `tools for`) | `GET /uts/v1/connections/<conn>/tools` |
| `tools validate <cat>/<tool> --data '<input>'` | `POST /uts/v1/tools/<cat>/<tool>/validate {"input":<input>}` |
| `tools exec <cat>/<tool> --connection <conn> --data '<input>'` | `POST /uts/v1/tools/<cat>/<tool>/execute {"connection_id":<conn>,"input":<input>}` |

Add `-o json` to any `connect` command for machine-readable output (agents get a
`_next` field).

## The input envelope

`input` is an object with up to four sub-keys; use only what the tool's schema
declares:

| Sub-key | Meaning |
| --- | --- |
| `path_params` | Values filling `{placeholders}` in the upstream path (e.g. `calendarId`). `null` if none. |
| `query_params` | URL query-string values. `null` if none. |
| `body` | Request body for write tools. |
| `headers` | Extra headers (rare). Auth is injected server-side — don't send credentials. |

The tool's `input.schema` constrains each field (patterns, `minLength`/`maxLength`,
`required`, enums). Read it before constructing a call:

```bash
connect tools show <category>/<tool> --examples
curl -s "$URL/openapi.json" | jq '.components.schemas.ToolExecuteRequest' # envelope schema
```

## Tool anatomy: schema + contract + errors

Each tool ships a typed bundle. The client-visible parts:

- **`manifest`** — identity (`name`, `id`, `version`, `stability`), `category`,
 `endpoint` (base_url/path/method), `capabilities` (`idempotent`, `safe`,
 `paginated`, `bulk`, `async`), and `requirements` (auth type + required
 `scope`, `rate_limit`).
- **`input.schema`** — JSON Schema for the envelope.
- **`output.schema`** — JSON Schema for the response (`status` enum + body shape).
- **`contract`** — `preconditions`, `postconditions`, `invariants`, declared
 `side_effects` (reads/writes/external_calls). This is the pre/postcondition
 layer that makes execute predictable.
- **`errors`** — the catalog of every error the tool can return.

## A worked example: slack/post-message

`GET /uts/v1/tools/slack/post-message` surfaces this shape (abridged):

**manifest** — `category: "messaging"`, method `POST /api/chat.postMessage`,
`capabilities.idempotent:false safe:false`, requires bearer auth with scopes
`["chat:write","chat:write.public"]`, rate limit `60 / 1m` per workspace.

**input.schema** — `body.channel` matches `^[CDG][A-Z0-9]+$|^[#@].+$`;
`body.text` is `minLength:1 maxLength:40000`; `blocks` is an array (`maxItems:50`).
(`path_params`/`query_params` are `null` — everything rides in `body`.)

**contract** — preconditions include `content_present` ("input.text exists OR
input.blocks exists"); postconditions include `message_posted`
("output.ok == true AND output.ts exists"); invariant "message content is never
modified by the service".

**errors** (branch on `code`, honor `retryable`/`retry_after`):

| status | code | category | retryable |
| --- | --- | --- | --- |
| 400 | `INVALID_CHANNEL` | validation | false |
| 400 | `MSG_TOO_LONG` | validation | false |
| 401 | `INVALID_AUTH` | authentication | false |
| 403 | `NOT_IN_CHANNEL` | authorization | false |
| 403 | `IS_ARCHIVED` | authorization | false |
| 429 | `RATE_LIMITED` | rate_limit | true (`retry_after: 60`) |
| 500 | `INTERNAL_ERROR` | server | true (exponential backoff) |
| 503 | `SERVICE_UNAVAILABLE` | availability | true (`retry_after: 30`) |

Validate then execute:

```bash
curl -X POST "$URL/uts/v1/tools/slack/post-message/validate" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"input":{"body":{"channel":"C123","text":"hi"}}}'

curl -X POST "$URL/uts/v1/tools/slack/post-message/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"conn_..","input":{"body":{"channel":"C123","text":"hi"}}}'
```

## Batch

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

Each request entry carries its own `category`/`tool`/`connection_id`/`input`;
results are returned per item.

## Why a tool over a raw proxy call

| | UTS tool (`/uts/v1/tools/..`) | Raw proxy (`/proxy/{conn}/<path>`) |
| --- | --- | --- |
| Input validation | Yes — against `input.schema`, plus a `validate` dry-run | No — provider rejects malformed calls at runtime |
| Errors | Structured catalog (`code`, `category`, `retryable`) | Raw provider error, shape varies |
| Contract | Pre/postconditions + declared side-effects | None |
| Discovery | `catalog` / `search` / `show` | You must know the provider's path |
| Coverage | First-class subset (Xero 33, Meta Ads 23, Google Ads 11, Slack/GitHub/…) | Everything, including un-wrapped endpoints |

**Prefer a tool when one exists.** For any endpoint not yet wrapped, the raw
passthrough `/<proxy>/{connection_id}/<path>` (`GET|POST|PUT|DELETE`) reaches it —
so there's no hard gap, just a typed fast path plus a universal fallback. Both go
through the same authenticated mesh; provider credentials and headers are injected
server-side either way.
