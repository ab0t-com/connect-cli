<p align="center">
  <img src="assets/hero.png" alt="connect — one authenticated gateway linking your app and your AI agents to every service" width="880">
</p>

<h1 align="center">connect</h1>

<p align="center"><em>one authenticated CLI for every service you connect</em></p>

---

`connect` is the client for the **ab0t integration mesh** — a hosted,
multi-tenant gateway that holds your connections to third-party services and
lets you act through them without ever handling their tokens yourself.

Connect Slack, GitHub, Google (Calendar / Sheets / Slides / Gmail / Drive),
Microsoft, Salesforce, Jira, Discord, Stripe, SendGrid, Twilio — and now
**Xero, Meta Ads, and Google Ads** — once. After that, every API call, tool
run, and webhook flows through one authenticated endpoint with credentials
injected and tenancy enforced server-side.

```
 connect ──HTTPS──► connect.service.ab0t.com ──► Slack / GitHub / Google /
 (you) (the mesh: proxy · tools · webhooks) Stripe / Xero / ..
 │
 └─ stored, encrypted connection credentials
 (you never paste a provider token into a call)
```

A **connection** is a stored credential for one account of one service. You
grant consent **once** (an OAuth browser flow, or a single API-key paste);
after that, acting through the connection is unlimited and headless — ideal
for scripts, CI, and AI agents.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/ab0t-com/connect-cli/main/install.sh | sh
```

The installer is POSIX sh, HTTPS-only, picks the right binary for your
platform (linux/macOS/Windows, amd64/arm64), verifies the published sha256
before installing, keeps the previous binary for one-step rollback, and is
idempotent. Pin a version or change the location:

```sh
REF=v0.1.0 curl -fsSL https://raw.githubusercontent.com/ab0t-com/connect-cli/main/install.sh | sh
PREFIX=$HOME/.local curl -fsSL ../install.sh | sh
```

Or grab a binary directly from [`release/`](release/) and verify it against
[`release/checksums.txt`](release/checksums.txt). The binaries are static —
no runtime dependencies.

## Quickstart

`connect` authenticates to the ab0t auth mesh. You can drive it interactively
(browser login) or headlessly (an API key minted in the mesh).

```sh
# 1. Authenticate. Browser flow for humans:
connect login
# …or fully headless (agents / CI):
export CONNECT_API_KEY=ab0t_sk_live_.. # a service/user key from the auth mesh
connect status

# 2. See what you can connect, and what's already connected.
connect services list # the connectable catalog
connect connections list # your connections (act through connection_id)

# 3. Connect a service.
connect add github # OAuth: opens a browser for one-time consent
connect add sendgrid --api-key SG.xxxx --name prod # API-key services connect headlessly

# 4. Act through the connection.
connect call <connection_id> GET /user/repos # raw API passthrough (proxy)
connect tools catalog # discover normalized tools
connect tools show google-calendar/create_event --examples

# 5. Receive events.
connect subscribe github push --connection <conn> --url https://myapp.com/hook --name deploy
connect webhooks deliveries list # debug deliveries
```

Not sure what to run next? `connect quickstart` reads your current auth +
connection state and prints **only the remaining steps**, each copy-pasteable.
`connect doctor` diagnoses config/auth/platform and ends with a single highest-
priority next action. Add `-o json` to any command for machine-readable output
(agents get a structured `_next` field instead of prose hints).

## The three surfaces

- **Proxy** — `connect call <conn> <METHOD> <path>` (or `connect proxy …`) is a
 transparent passthrough to the provider's own API. The mesh injects the
 upstream credential, refreshes OAuth tokens on demand, applies rate limits,
 and audits the call. You write the provider's native API path; you never
 handle its secret.
- **Tools (UTS)** — `connect tools …` exposes each provider's operations as
 normalized, input-validated "tools" (`category/tool/execute`). Prefer tools
 when they exist: structured input schemas, validation, and richer errors than
 raw proxy. Great for agents.
- **Webhooks** — `connect subscribe …` registers a callback; the mesh receives
 provider events, matches them to your subscriptions, and delivers them to
 your URL — HMAC-signed (`X-Webhook-Signature`), retried with backoff, and
 fully replayable/observable (`connect webhooks deliveries …`).

## Connection model — consent vs. action

| | What it is | How often |
| --- | --- | --- |
| **Consent** | Creating a connection (OAuth browser approval, or one API-key paste) | **Once** per account |
| **Action** | Proxy calls, tool runs, webhook routing through that `connection_id` | **Unlimited & headless** |

OAuth tokens can never be injected via the API-key path — reserved keys
(`access_token`, `refresh_token`, …) are stripped server-side by design. OAuth
is a consent flow only. For fully agentic / headless OAuth (service accounts,
device-code, pre-provisioning) see
[`skills/connect/references/agentic-connections.md`](skills/connect/references/agentic-connections.md).

## Public service

- **API base:** `https://connect.service.ab0t.com` (also reachable as
 `https://connect.service.ab0t.com`).
- **OpenAPI:** `GET /openapi.json` is the source of truth for request/response
 shapes. LLM index at `GET /llm.txt`.

## Repository layout

```
connect/
├── install.sh # POSIX, checksum-verified, multi-platform installer (curl | sh)
├── release/ # per-platform binaries + checksums.txt + VERSION
├── docs/ # USAGE (full CLI reference) · PROVIDERS (the connectable catalog)
├── prompts/ # system prompt for driving connect with an LLM
├── skills/ # portable agent skills (see below) — drop into any SKILL.md-aware agent
└── llms.txt # LLM-readable bootstrap index for this tool
```

- [`docs/USAGE.md`](docs/USAGE.md) — the connection model, OAuth/API-key, proxy,
 tools, webhooks, headless/agent auth — every command.
- [`docs/PROVIDERS.md`](docs/PROVIDERS.md) — the public catalog of connectable
 services and what you can do with each.
- [`prompts/AGENT_SYSTEM_PROMPT.md`](prompts/AGENT_SYSTEM_PROMPT.md) — drop-in
 system prompt for an agent that operates connect.
- [`skills/`](skills/) — **11 portable agent skills** (Agent Skills standard — work in
 Claude Code, Codex, Gemini, opencode, …). Start with `connect-concepts`, then pick the surface you need:

 *Understand the system*
 - [`connect-concepts/`](skills/connect-concepts/) — **read first**: the mental model (org/tenant, connections, the three surfaces, consent vs action) + glossary.
 - [`connect-security/`](skills/connect-security/) — security & data handling (encryption at rest, consent, signature verify, rotation, tenant isolation).

 *Drive the CLI / the surfaces*
 - [`connect/`](skills/connect/) — the daily-driver: connection model + the three surfaces.
 - [`connect-connections/`](skills/connect-connections/) — how connections work & first setup (OAuth vs API-key, multi-tenant, `connection_id`).
 - [`connect-proxy/`](skills/connect-proxy/) — call a provider's own API through the authenticated proxy.
 - [`connect-tools/`](skills/connect-tools/) — discover & run the validated UTS tools.
 - [`connect-realtime/`](skills/connect-realtime/) — receive events: webhooks (HTTP push) and websockets (live stream).
 - [`connect-auth/`](skills/connect-auth/) — CLI auth & profiles (login, API keys, dev/prod, headless).

 *Build & operate*
 - [`connect-recipes/`](skills/connect-recipes/) — end-to-end cookbook (Slack post, Xero sync, live GitHub stream, webhook pipeline, multi-tenant SaaS, agents).
 - [`connect-troubleshooting/`](skills/connect-troubleshooting/) — unified error reference (every code × surface → fix) + rate limits + debug playbook.
 - [`connect-rest-api/`](skills/connect-rest-api/) — call the public REST API directly (curl/Python/JS/Go), no CLI.

 The live `https://connect.service.ab0t.com/openapi.json` is the authoritative REST contract.

## License

MIT — see [`LICENSE`](LICENSE).
