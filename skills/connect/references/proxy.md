# API Proxy — call a connected service's own API

The proxy forwards your request to the upstream provider with **credentials injected server-side** and **tenancy enforced**. You never see or hold the provider token. You address the provider's *native* API path.

## Contents
- [Shape](#shape)
- [CLI](#cli)
- [Raw API](#raw-api)
- [Proxy vs UTS — which to use](#proxy-vs-uts--which-to-use)
- [Errors](#errors)
- [Patterns](#patterns)

## Shape
```
{METHOD} {api_url}/proxy/{connection_id}/{provider_native_path}
```
- `{connection_id}` selects the account + injects its credential.
- `{provider_native_path}` is the upstream API path verbatim (e.g. `/calendar/v3/users/me/calendarList` for Google Calendar, `/v3/marketing/contacts` for SendGrid).
- Query string, request body, and most headers pass through. Auth headers are added by the proxy — do not send the provider's token yourself.

## CLI
```bash
# call = convenience verb (METHOD as an arg)
connect call <connection_id> GET /v3/marketing/contacts
connect call <connection_id> GET '/calendar/v3/users/me/calendarList'

# proxy = METHOD + connection/path joined, with a body
connect proxy POST <connection_id>/calendar/v3/calendars/primary/events \
 --data '{"summary":"Sync","start":{"dateTime":"2026-09-01T10:00:00Z"},"end":{"dateTime":"2026-09-01T10:30:00Z"}}'

connect proxy GET <connection_id>/info # connection metadata (no upstream call)
connect call <connection_id> GET /v3/x --output json # raw JSON for scripting
```

## Raw API
```bash
curl -X POST "$URL/proxy/$CONN/calendar/v3/calendars/primary/events" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"summary":"Sync","start":{..},"end":{..}}'

# metadata only:
curl "$URL/proxy/$CONN/info" -H "X-API-Key: $KEY"
```
The response is the upstream provider's response (status, body) passed back through.

## Proxy vs UTS — which to use
| | Proxy (`/proxy/..`) | UTS tools (`/uts/v1/tools/..`) |
|---|---|---|
| Path | provider's **native** API path | normalized `category/tool` |
| Shape | provider's own request/response | unified envelope, consistent across services |
| Use when | you know the provider's API / need an endpoint UTS doesn't expose | you want one schema across many providers (agents) |
| Discovery | provider's own docs | `connect tools catalog` / `tools show` |

Rule of thumb: **agents prefer UTS** (normalized, discoverable, validated). **Reach for the proxy** for full provider API coverage or one-off endpoints. See [uts-tools.md](uts-tools.md).

## Errors
- `404` — connection_id doesn't exist (or not yours).
- `403` — connection exists but belongs to another tenant (cross-tenant blocked).
- `409`-style "Connection is not active" — re-connect / the credential was revoked upstream.
- `401`/`4xx`/`5xx` with an upstream body — that's the **provider's** error, passed through; debug against the provider, not ab0t.

## Patterns
- **Pagination/rate limits are the provider's** — honor their `Link`/cursor/`Retry-After`; the proxy doesn't auto-paginate.
- **Discover the connection first**: `connect connections list` → pick the `connection_id` → `proxy GET <id>/info` to confirm it's active before a real call.
- **Multiple accounts** of one service = multiple connections; pick the right `connection_id`.
- **Batch** several proxy calls in one request: `connect batch <conn> --request 'GET /a' --request 'GET /b'` (see `connect batch --help`).
