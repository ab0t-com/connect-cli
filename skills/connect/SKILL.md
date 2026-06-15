---
name: connect
description: Use the connect CLI to drive the ab0t integration mesh — connect third-party services (Slack/GitHub/Google/SendGrid/Stripe…), call their APIs through one authenticated proxy, execute normalized UTS tools, and route/observe webhooks. Use when wiring an app or AI agent to external services via ab0t, connecting a service (OAuth or API key), running a tool against a connected account, proxying an API call, subscribing to or verifying webhook signatures, building a webhook receiver endpoint, or designing headless/agentic connection flows. Covers the connection model (consent vs action), headless auth for agents, and the proxy/tool/webhook surfaces.
---

# connect — the ab0t integration mesh CLI

connect is the client for the connect mesh (`connect.service.ab0t.com`). The service is a **multi-tenant integration mesh** with three surfaces:
- **Proxy** — call a connected service's API through one authenticated gateway (creds injected, tenancy enforced).
- **UTS tools** — normalized `category/tool/execute` over all services (and internal mesh tenants).
- **Webhooks** — inbound provider events → matched to subscriptions → delivered to your endpoint (HMAC-signed, retried) → watchable.

The live `GET /openapi.json` is the source of truth for request/response shapes.

## Quick start (agent/CI — API key, no browser)
```bash
export CONNECT_API_KEY=ab0t_sk_live_.. # service/user key from the auth mesh
connect status --api-url https://connect.service.ab0t.com
connect services list # 13+ services
connect connections list # your connections (connection_id is what you act through)
```
Config lives in `~/.config/connect/config.yaml` (api_url, auth_url, org_slug) + per-profile `credentials-<profile>.json`. Override per command with `--api-url`, `--profile`, `--output json`.

## The connection model (READ THIS for agentic use)
A **connection** is a stored, encrypted credential for one account of one service, owned by a user/org. **Everything you DO (proxy, tools, webhooks) acts through a `connection_id`.** Creating a connection = granting consent ONCE; acting through it is unlimited and headless.

Two ways to create a connection:
- **API-key services** (SendGrid, Stripe, Twilio) — fully headless:
 ```bash
 connect add sendgrid --api-key SG.xxxx --name "prod"
 # = POST /connections/sendgrid/api-key {"api_key":"SG.xxxx","name":"prod"}
 ```
- **OAuth services** (Google, Slack, GitHub, Jira, Discord…) — require **one-time human consent** (browser):
 ```bash
 connect add github # opens browser → consent → connection created
 connect add github --oauth # force OAuth if auto-detect is unsure
 ```
 Under the hood: `POST /connections/{svc}/oauth/authorize {redirect_uri}` → authorization URL → human approves → provider redirects to `/connections/{svc}/oauth/callback` → tokens exchanged + stored. The CLI opens the browser and polls for the new connection (~5 min timeout).

**You cannot inject OAuth tokens via the api-key endpoint** — reserved keys (`access_token`, `refresh_token`, …) are stripped server-side by design. OAuth = consent flow only.

**For fully agentic / headless OAuth**, see **[references/agentic-connections.md](references/agentic-connections.md)** — consent-vs-action split, service accounts, device-code, pre-provisioning, and exact code.

## Act through a connection
```bash
# Proxy: call the upstream API directly (path is the provider's API path)
connect call <connection_id> GET /calendar/v3/users/me/calendarList
connect proxy POST <connection_id>/calendar/v3/calendars/primary/events --data '{..}'
# = POST /proxy/{connection_id}/calendar/v3/calendars/primary/events

# UTS tool (normalized): execute a tool against the connection
curl -X POST $URL/uts/v1/tools/google-calendar/create_event/execute \
 -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
 -d '{"connection_id":"conn_..","input":{"path_params":{"calendarId":"primary"},
 "body":{"summary":"Hi","start":{"dateTime":"2026-09-01T10:00:00Z"},"end":{"dateTime":"2026-09-01T10:30:00Z"}}}}'
connect tools catalog # discover tools by category
connect tools show google-calendar/create_event --examples
```
Wrong-type connection → `400`; missing connection → `404`; cross-tenant → `403`. Tenancy is enforced — you only act on your own connections.
Deep dives: **[references/proxy.md](references/proxy.md)** (native-API passthrough, proxy-vs-UTS) · **[references/uts-tools.md](references/uts-tools.md)** (the input envelope, discovery, validate, agentic loop).

## Webhooks (events in → delivered to you)
```bash
connect subscribe github push --connection <conn> --url https://myapp.com/hook --name deploy
# prints the receiver URL (paste into the provider) + a signing secret (shown once)
connect webhooks subscriptions list # NOTE: nested under `webhooks` (see Commands)
connect webhooks deliveries list # debug deliveries
connect webhooks deliveries show <del_id> --attempts
connect webhooks deliveries retry <del_id>
connect tail # real-time stream (see caveat below)
```
Every delivery is signed: `X-Webhook-Signature: t=<ts>,v1=<HMAC-SHA256(raw body, signing_secret)>`, plus `X-Webhook-Id`, `X-Webhook-Timestamp`. **`v1` is HMAC of the raw body ONLY — `t` is not signed (not Stripe-style).** Failed deliveries retry with backoff.
Deep dive (verifier code in Python/JS/Go, a receiver endpoint, filtering, dedupe): **[references/webhooks.md](references/webhooks.md)**.

## Onboarding & next-step guidance
- **`connect quickstart`** — state-aware onboarding ladder. Reads local auth state + your connections and prints **only the remaining steps** (login → connect a service → subscribe / act → all set), each a copy-pasteable command. It prescribes `login` when logged out and `connect` (not `login`) when logged in with zero connections. If the platform is unreachable it degrades to the full static ladder instead of hanging. It is its own verb — distinct from `profile use` (profile switching) and `use` (default connection).
- **`connect doctor`** — diagnoses config/auth/platform/connections and ends with a single highest-priority NEXT line (the same state-aware action quickstart would prescribe).

### stderr-prose vs stdout `_next` contract (READ for agents)
Two audiences, two channels:
- **Humans** get colorized prose NEXT hints on **stderr** (via `output.Hint`), so they never pollute stdout. Pipes/`-o json` stdout stay clean.
- **Agents** running `-o json` (or `quickstart --agent`) get **no prose at all** — instead a machine-readable **`_next`** field inside the stdout JSON envelope:
 ```json
 { "_next": { "state": "no_connections",
 "next_command": "connect add <service>",
 "why": "You're signed in but have no service connections yet.",
 "alternatives": ["connect services list (browse connectable services)"] } }
 ```
 `quickstart -o json` emits `{ "_next": {…} }`; `doctor -o json` emits `{ "checks": [{name,status,fix}…], "_next": {…} }`. Parse `_next.next_command` to drive the agent forward. Suppress hints entirely with `--no-hints` / `CONNECT_NO_HINTS`.

## Commands
Top-level: `auth login/logout/whoami/token`, `status`, `quickstart`, `services list/show`, `connections list/show/delete`, `connect`, `subscribe`, `proxy`/`call`, `info`, `tools …`, `batch`, `tail`, `listen`, `doctor`, `examples`, `version`.
Webhook management is **nested**: `connect webhooks subscriptions|events|deliveries|receiver-url …` (only `subscribe`, `tail`, `listen` have top-level shortcuts).

## Shell completion
Dynamic TAB-completion for service ids, connection ids, tool keys, categories, profiles, subscription/event/delivery ids, HTTP methods, `--status` values, and `-o` values — across every command, including the `subs`/`events`/`deliveries`/`conns` aliases. Cache-first + fail-open: 1.5s network cap, never hangs, returns nothing offline/logged-out. One-step install:
```bash
connect completion install # auto-detect shell, write + show how to activate
connect completion install --shell zsh # force a shell
```
Manual install:
```bash
connect completion zsh > "${fpath[1]}/_connect" # zsh (or: source <(connect completion zsh))
connect completion bash | sudo tee /etc/bash_completion.d/connect # bash
connect completion fish > ~/.config/fish/completions/connect.fish # fish
```
**Completion binds to the command word `connect`, not `./connect`** — invoking by path gives zero completion. Put the binary on PATH and call it as `connect`: `cp connect ~/.local/bin/connect`. Cache lives at `~/.config/connect/completion-cache-<profile>.json` (0600, ids/names only — never secrets); agents driving a shell can read it for valid ids. Event-type completion is a best-effort hint for well-known providers — use `connect services events <service>` for the authoritative list.

## Reference files (load on demand)
- **[references/agentic-connections.md](references/agentic-connections.md)** — connecting services headlessly: consent-vs-action, service accounts, device-code, pre-provisioning, agent connect loop.
- **[references/proxy.md](references/proxy.md)** — the API proxy: native-path passthrough, CLI + raw API, proxy-vs-UTS decision, error map.
- **[references/uts-tools.md](references/uts-tools.md)** — the UTS tool-calling network: discovery, the `input` envelope (path_params/query_params/body), validate, agentic loop.
- **[references/webhooks.md](references/webhooks.md)** — webhooks end-to-end: subscribe, **signature verification (Python/JS/Go)**, receiver endpoint, filtering, retries/dedupe, observe/debug.

For the raw REST API without the CLI, the live OpenAPI contract is published at `https://connect.service.ab0t.com/openapi.json`.

## Gotchas
- Default `api_url` is `https://connect.service.ab0t.com` (prod); the dev mesh is `https://connect.dev.ab0t.com`. Check the active profile's host with `connect profile show`; override per-command with `--api-url` or `CONNECT_API_URL`.
- `subscriptions test` needs a body: `{"event_type":"<type>"}`.
- Delivery IDs are `del_<hex>` (not UUIDs); subscription/event IDs are UUIDs.
- Real-time `tail` is reliable only when the service runs its delivery in-process; in lambda-delivery mode delivery-status may not stream (infra-dependent).
