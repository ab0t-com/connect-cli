# Usage — the connection model, the three surfaces, every command

Day-to-day reference for driving the ab0t integration mesh with `connect`.
The provider catalog is in [`PROVIDERS.md`](PROVIDERS.md); the live
`GET /openapi.json` is the authoritative contract for request/response shapes.

## Mental model

```
 connect ──HTTPS──► connect.service.ab0t.com ──► Slack / GitHub / Google /
 (you) (proxy · tools · webhooks) Stripe / Xero / Meta Ads / ..
 │
 └─ encrypted connection credentials (per account, per org)
```

- A **connection** is a stored, encrypted credential for **one account of one
 service**, owned by your user/org. It is identified by an opaque
 `connection_id`.
- **Everything you DO acts through a `connection_id`** — proxy calls, tool runs,
 webhook subscriptions.
- **Creating a connection = granting consent ONCE.** Acting through it
 afterward is unlimited and headless.
- **Tenancy is enforced server-side.** You only ever see and act on your own
 org's connections. Wrong-type connection → `400`; missing → `404`;
 another tenant's connection → `403`.

## Configuration & profiles

Config lives at `~/.config/connect/config.yaml` (`api_url`, `auth_url`,
`org_slug`) with per-profile `credentials-<profile>.json`. Override per command:

| Flag / env | Effect |
| --- | --- |
| `--api-url <url>` | Point at a specific mesh (default `https://connect.service.ab0t.com`) |
| `--profile <name>` | Use a named profile's credentials |
| `-o json` / `--output json` | Machine-readable output (agents get a `_next` field) |
| `--no-hints` / `CONNECT_NO_HINTS` | Suppress the human-facing NEXT prose on stderr |
| `CONNECT_API_KEY` | Headless auth (an `ab0t_sk_live_…` mesh key) |

The default `api_url` is `https://connect.service.ab0t.com` (also reachable
as `https://connect.service.ab0t.com`). If a call 404s with an older profile,
check `connect profile show` and re-point, or pass `--api-url` explicitly.

## Authenticating

Two paths — pick by audience.

**Humans (browser):**

```bash
connect login # opens the auth-mesh hosted login, stores credentials
connect auth whoami # confirm identity / org
connect logout
```

**Agents / CI (headless, no browser):**

```bash
export CONNECT_API_KEY=ab0t_sk_live_.. # a service/user key minted in the auth mesh
connect status # confirms the key reaches the mesh
```

The API key never needs a browser. Authenticating to the mesh is orthogonal to
connecting third parties — you can be logged in and still have zero connections.

## Onboarding helpers

```bash
connect quickstart # state-aware ladder: prints ONLY your remaining steps
connect doctor # diagnoses config/auth/platform/connections + one NEXT action
```

`quickstart` reads your local auth state and your connections and prescribes the
next concrete command (login → connect → act → done). With `-o json` (or
`quickstart --agent`) it emits a machine-readable `_next` envelope:

```json
{ "_next": { "state": "no_connections",
 "next_command": "connect add <service>",
 "why": "You're signed in but have no service connections yet.",
 "alternatives": ["connect services list (browse connectable services)"] } }
```

`doctor -o json` adds a `checks` array (`{name, status, fix}`) alongside `_next`.

## Discovering services

```bash
connect services list # the connectable catalog (13+ services)
connect services show github # one service: auth type, scopes, webhook support
connect services events github # the authoritative event-type list for webhooks
```

See [`PROVIDERS.md`](PROVIDERS.md) for the human-readable catalog and what each
provider lets you do.

## Connecting a service (granting consent — once)

### API-key services — fully headless

For services authed by a static secret (SendGrid, Stripe, Twilio, …):

```bash
connect add sendgrid --api-key SG.xxxx --name prod
# = POST /connections/sendgrid/api-key {"api_key":"SG.xxxx","name":"prod"}
# -> connection_id you act through
```

An agent that holds the secret creates the connection with zero human
interaction.

### OAuth services — one-time human consent

For services that grant access to a user's data (Google, Slack, GitHub, Jira,
Discord, Microsoft, Salesforce, Xero, Meta Ads, Google Ads, …):

```bash
connect add github # auto-detect; opens a browser for one-time consent
connect add github --oauth # force OAuth if auto-detect is unsure
```

Under the hood: `POST /connections/{svc}/oauth/authorize {redirect_uri}` →
authorization URL → a human approves the requested scopes → provider redirects
to `/connections/{svc}/oauth/callback` → tokens exchanged and stored. The CLI
opens the browser and polls for the new connection (~5 min timeout).

**You cannot inject OAuth tokens via the api-key endpoint** — reserved keys
(`access_token`, `refresh_token`, …) are stripped server-side by design. OAuth
is a consent flow only.

### Managing connections

```bash
connect connections list # all your connections (connection_id, service, status)
connect connections show <connection_id> # detail
connect connections delete <connection_id>
connect use <connection_id> # set a default connection for subsequent commands
```

## Surface 1 — Proxy (call the provider's own API)

`connect call` / `connect proxy` is a transparent passthrough. You write the
**provider's native API path**; the mesh injects the upstream credential,
refreshes OAuth tokens on demand, applies rate limits, and audits the call.

```bash
# GET passthrough
connect call <connection_id> GET /user/repos?per_page=100

# POST with a body
connect proxy POST <connection_id>/calendar/v3/calendars/primary/events \
 --data '{"summary":"Standup","start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}'
# = POST /proxy/{connection_id}/calendar/v3/calendars/primary/events

# Connection metadata without making the upstream call
connect info <connection_id>
```

The HTTP method on **your** call is the method used upstream. The proxy filters
out hop-by-hop and auth headers (it replaces the auth header with the stored
credential). Response headers include `X-Integration-Service`,
`X-Integration-Rate-Limit`, and `X-Integration-Cost` when applicable.

Deep dive: [`skills/connect/references/proxy.md`](./skills/connect/references/proxy.md).

## Surface 2 — Tools (UTS: normalized, validated operations)

UTS (Unified Tool Schema) exposes each provider's operations as named tools
with validated input schemas. **Prefer tools over raw proxy when one exists** —
input validation, structured errors, and ownership checks are richer.

```bash
connect tools catalog # discover tools by category
connect tools search calendar # search the catalog
connect tools show google-calendar/create_event --examples
```

Execute via the API (the input envelope is `path_params` / `query_params` /
`body`):

```bash
curl -X POST "$URL/uts/v1/tools/google-calendar/create_event/execute" \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"conn_..",
 "input":{"path_params":{"calendarId":"primary"},
 "body":{"summary":"Hi",
 "start":{"dateTime":"2026-09-01T10:00:00Z"},
 "end":{"dateTime":"2026-09-01T10:30:00Z"}}}}'
```

`connection_id` ownership and org boundary are enforced server-side. Validate
without executing via `POST /uts/v1/tools/{category}/{tool}/validate`.

Deep dive: [`skills/connect/references/uts-tools.md`](./skills/connect/references/uts-tools.md).

## Surface 3 — Webhooks (provider events delivered to you)

```bash
connect subscribe github push --connection <conn> --url https://myapp.com/hook --name deploy
# prints the receiver URL (paste into the provider) + a signing secret (shown ONCE)

connect webhooks subscriptions list # management is nested under `webhooks`
connect webhooks deliveries list # outbound deliveries to your URL
connect webhooks deliveries show <del_id> --attempts
connect webhooks deliveries retry <del_id>
connect tail # real-time stream (top-level shortcut)
```

Only `subscribe`, `tail`, and `listen` have top-level shortcuts; everything else
lives under `connect webhooks subscriptions|events|deliveries|receiver-url …`.

### Verifying a delivery

Every delivery is HMAC-signed:

```
X-Webhook-Signature: t=<unix_ts>,v1=<HMAC-SHA256(raw_body, signing_secret)>
X-Webhook-Id: <delivery id>
X-Webhook-Timestamp: <unix_ts>
```

`v1` is HMAC of the **raw request body only** (lowercase hex) — `t` is **not**
part of the signed payload (this is *not* the Stripe `t.body` scheme). Verify by
recomputing `HMAC_SHA256(key=signing_secret, msg=raw_body_bytes)` and
constant-time comparing to `v1`. Failed deliveries retry with backoff. Verifier
code in Python/JS/Go and a full receiver endpoint are in
[`skills/connect/references/webhooks.md`](./skills/connect/references/webhooks.md).

## Headless / agentic connections

The rule: **separate CONSENT (one-time, identity-granting) from ACTION
(repeated, headless).** A connection is the durable consent artifact; once it
exists, an agent acts through `connection_id` forever with no human in the loop.

- **API-key services** — an agent holding the secret self-connects. Done.
- **OAuth user-data services** — one human consent per account. In an agent
 loop, emit the `authorization_url`, a human (HITL) approves, the agent
 resumes on the new `connection_id`. Never re-consent if a valid connection
 already exists — `GET /connections/` and reuse.

The full pattern table (operator pre-provision, service-account/domain-wide
delegation, device-code, client-credentials) and a minimal agent connect loop
are in
[`skills/connect/references/agentic-connections.md`](./skills/connect/references/agentic-connections.md).

## Shell completion

Dynamic TAB-completion for service ids, connection ids, tool keys, categories,
profiles, subscription/event/delivery ids, HTTP methods, and `-o` values —
cache-first and fail-open (never hangs; returns nothing offline/logged-out).

```bash
connect completion install # auto-detect shell, write + show activation
connect completion install --shell zsh # force a shell
```

**Completion binds to the command word `connect`, not `./connect`** — put the
binary on PATH and call it as `connect`. The completion cache lives at
`~/.config/connect/completion-cache-<profile>.json` (mode 0600, ids/names only —
never secrets).

## Command reference

| Group | Commands |
| --- | --- |
| Auth | `login`, `logout`, `auth whoami`, `auth token`, `status` |
| Onboarding | `quickstart`, `doctor` |
| Catalog | `services list`, `services show <svc>`, `services events <svc>` |
| Connections | `connect <svc> [--api-key … \| --oauth]`, `connections list/show/delete`, `use <conn>` |
| Proxy | `call <conn> <METHOD> <path>`, `proxy <METHOD> <conn>/<path> [--data …]`, `info <conn>` |
| Tools | `tools catalog`, `tools search <q>`, `tools show <cat>/<tool> [--examples]`, `batch` |
| Webhooks | `subscribe …`, `tail`, `listen`, `webhooks subscriptions\|events\|deliveries\|receiver-url …` |
| Misc | `completion install`, `examples`, `version` |

## Common 4xx patterns

| Code | Likely cause | Fix |
| --- | --- | --- |
| `401` | Missing/expired token, or the key's `aud` doesn't match this service | Re-`login` or refresh the API key; check `connect status` |
| `401` on a proxy call | The connection's stored upstream credential expired (OAuth not refreshable) | `connect connections show <conn>`; re-do OAuth, or update the API-key connection |
| `400` | Wrong connection type for the operation | Use a connection of the right service |
| `403` | Tenant boundary — that `connection_id` belongs to another org | `connect connections list` to find your own |
| `404` on a tool | Tool not in the catalog for your version | `connect tools catalog`; check exact `category/tool` names |
| `422` on tool execute | Input failed schema validation | Run `/validate` first, or `connect tools show <cat>/<tool>` for the schema |
| `429` | Provider rate limit (mediated by the mesh) | The `X-Integration-Rate-Limit` response header shows current state |

## Gotchas

- Older profiles pinned to the retired `connect.dev.ab0t.com` host will 404 —
 re-point to `https://connect.service.ab0t.com` (or
 `connect.service.ab0t.com`).
- Delivery IDs are `del_<hex>` (not UUIDs); subscription/event IDs are UUIDs.
- `subscriptions test` needs a body: `{"event_type":"<type>"}`.
- Real-time `tail` is reliable only when the service runs delivery in-process;
 in some delivery modes the status stream is infra-dependent.

For driving connect with an LLM, see
[`prompts/AGENT_SYSTEM_PROMPT.md`](./prompts/AGENT_SYSTEM_PROMPT.md).
