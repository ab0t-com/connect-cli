# UTS — the tool-calling network

UTS (Unified Tool Schema) is a **normalized `category/tool/execute` layer** over every connected service. One consistent envelope — validated input, consistent errors — across Slack/Google/GitHub/etc. **and internal mesh tenants** (other ab0t services are reachable as "third parties" through the same surface). This is the layer agents should prefer.

## Contents
- [Discover tools](#discover-tools)
- [Execute a tool](#execute-a-tool)
- [The input envelope](#the-input-envelope)
- [Validate before executing](#validate-before-executing)
- [Errors](#errors)
- [Agentic loop](#agentic-loop)

## Discover tools
```bash
connect tools catalog # all tools, grouped by category
connect tools available <connection_id> # tools usable for a connection (alias: tools for)
connect tools show <category>/<tool> --examples
connect tools search <keyword>
```
LLM-readable indexes (great first stop for an agent): `GET /uts/llm.txt`, `GET /uts/v1/llm.txt`, `GET /services/{service_id}/llm.txt`.

## Execute a tool
```bash
# CLI
connect tools exec google-calendar/create_event --connection <conn> \
 --data '{"path_params":{"calendarId":"primary"},
 "body":{"summary":"Hi","start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}}'
```
```bash
# Raw API
curl -X POST "$URL/uts/v1/tools/google-calendar/create_event/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"conn_..","input":{"path_params":{"calendarId":"primary"},
 "body":{"summary":"Hi","start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}}}'
```
Request body is exactly four fields: `category`, `tool`, `connection_id`, `input` (an object). The URL carries `category`/`tool`; the body carries `connection_id` + `input`.

## The input envelope
`input` is a structured object — NOT a flat blob. Sub-keys (use what the tool's schema needs):
- `path_params` — values that fill `{placeholders}` in the upstream path (e.g. `calendarId`).
- `query_params` — URL query string values.
- `body` — the request body for write tools.
- `headers` — extra headers (rare).

Always check the tool's schema for the exact shape:
```bash
connect tools show google-calendar/create_event --examples
curl -s "$URL/openapi.json" | jq '.components.schemas.ToolExecuteRequest' # envelope schema
```

## Validate before executing
```bash
# POST /uts/v1/tools/{category}/{tool}/validate with {"input": {..}}
connect tools validate google-calendar/create_event --data '{"path_params":{..},"body":{..}}'
```
Returns whether `input` satisfies the tool's schema — cheaper than a failed execute, useful in agent loops before committing a side-effecting call.

## Errors
- `400` — invalid input (failed schema validation) OR connection type doesn't match the tool's category.
- `404` — unknown tool, or missing connection.
- `403` — connection belongs to another tenant.
- `422` — missing/ill-formed `connection_id`/`input` in the request body.
Tenancy is enforced: you only execute against your own connections.

## Agentic loop
```python
# 1. discover (or read /uts/llm.txt)
tools = GET(f"/uts/v1/tools?category=google-calendar")
# 2. find a connection (reuse; never re-consent if one is active)
conn = next(c for c in GET("/connections/") if c["service_id"]=="google_calendar" and c["status"]=="active")
# 3. (optional) validate, then execute — headless, forever
POST(f"/uts/v1/tools/google-calendar/create_event/execute",
 {"connection_id": conn["connection_id"],
 "input": {"path_params": {"calendarId": "primary"}, "body": {..}}})
```
Connection creation (consent) is the only step that may need a human, and only for OAuth services — see [agentic-connections.md](agentic-connections.md). Everything after is normalized and headless.
