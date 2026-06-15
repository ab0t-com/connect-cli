---
name: connect-connections
description: How CONNECTIONS work in the ab0t Connect integration mesh and how to set one up — the foundational connection model. Use when a user wants to "connect a service", "set up a connection", "link my Slack/Google/GitHub/Stripe account", asks "how do connections work", "OAuth vs API key connection", "what is a connection_id", "list my connections", "revoke / delete / disconnect a connection", "show connection status", or asks about multi-tenant connections (one org holding many users' connections, tenant_id / tenant_app_id), consent vs action, headless/agentic connecting, or which auth type a service uses. Start here before proxy calls, tool runs, or webhooks — every one of those acts through a connection_id created here.
---

# Connections — the ab0t Connect mesh

The Connect mesh CLI is `connect`. Public host: `connect.service.ab0t.com` (dev: `connect.dev.ab0t.com`).

A **connection** pairs three things: **your org** + **one service** + **stored, encrypted credentials for one account of that service**. It is identified by an opaque `connection_id`. Everything you DO afterward — proxy calls, tool runs, webhook subscriptions — acts **through that `connection_id`**. Creating the connection is the one-time consent; acting through it is unlimited and headless.

This serves three users: the **app builder** wiring connections for their own end-users, the **AI agent** connecting and acting headlessly, and the **ops owner** managing the org's fleet of connections.

## Quick start

```bash
# Auth to the mesh (orthogonal to connecting third parties)
export CONNECT_API_KEY=ab0t_sk_live_.. # agent/CI: a mesh key, no browser
connect status # confirm the key reaches the mesh
# or, for a human: connect login

connect services list # what's connectable (auth type per service)
connect services show github # OAuth or API key? scopes? webhooks?

# Connect (see the two paths below), then list / act / revoke:
connect connections list # your connections: connection_id, service, status
connect connections show <connection_id> # detail
connect connections delete <connection_id> # revoke
```

Authenticating to the mesh is not the same as having connections — you can be logged in with zero connections. Run `connect quickstart` for a state-aware "what's my next step" ladder.

## The connection model

- **One connection = one account of one service, owned by your org.** It holds encrypted credentials you never see; the API only ever returns `{"status":"encrypted"}` plus metadata.
- **Consent vs action.** Creating a connection grants consent once. After that, an agent acts through `connection_id` forever — no human in the loop.
- **Org-scoped and multi-tenant.** Connections belong to your org; you only ever see and act on your own org's connections. One org can hold **many** connections — many users, many accounts, many services. Server-side tenancy is enforced: a `connection_id` from another org returns `403`. See [references/multi-tenant.md](references/multi-tenant.md).

## Two ways to create a connection

The path is decided by the service's auth type (`connect services show <svc>`). You cannot pick — an OAuth service rejects the api-key path (`400`) and vice versa.

### API-key services — fully headless

For services authed by a static secret (SendGrid, Stripe, Twilio, …). An agent that holds the secret self-connects with zero human interaction:

```bash
connect add sendgrid --api-key SG.xxxx --name prod
# = POST /connections/sendgrid/api-key {"api_key":"SG.xxxx","name":"prod"}
# -> a connection_id you act through
```

The same with raw HTTP:

```bash
curl -X POST "$URL/connections/sendgrid/api-key" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' -d '{"api_key":"SG.xxxx","name":"prod"}'
```

Service-specific extras (e.g. `region`, `account_sid`) go in `additional_config` — but credential/routing keys (`access_token`, `refresh_token`, `tenant_id`, …) are **reserved and rejected**. You cannot smuggle OAuth tokens through this endpoint.

### OAuth services — one-time human consent

For services that grant access to a user's data (Google, Slack, GitHub, Jira, Discord, Microsoft, Salesforce, Xero, Meta Ads, Google Ads, …). One human must approve scopes — there is no way around the approval itself; only *how* it's collected varies.

```bash
connect add github # auto-detect; opens a browser for one-time consent
connect add github --oauth # force OAuth if auto-detect is unsure
```

Under the hood, the flow is two calls plus a human approval:

```bash
# 1. ask the service for an authorization URL
curl -X POST "$URL/connections/github/oauth/authorize" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"redirect_uri":"https://connect.service.ab0t.com/connections/github/oauth/callback"}'
# -> { "authorization_url": "https://github.com/login/oauth/authorize?..", "state":"..", "expires_in":300 }

# 2. a HUMAN opens authorization_url and approves scopes (the only human step)
# 3. the provider redirects to /connections/github/oauth/callback;
# the mesh exchanges code -> tokens, stores them, creates the connection
# 4. the connection now appears in your list
curl "$URL/connections/" -H "X-API-Key: $KEY" | jq '.[]|select(.service_id=="github")'
```

`connect add` automates all four steps (opens the browser, polls ~5 min for the new connection). For an agent loop, emit the `authorization_url`, let a human approve, then resume on the new `connection_id`. **Never re-consent if a valid connection already exists** — list first and reuse. Full headless/agentic patterns (operator pre-provision, service accounts, device-code, the connect loop) live in the `connect` skill's `references/agentic-connections.md`.

## Acting through a connection

Once you hold a `connection_id`, hand it to any of the three surfaces:

```bash
connect call <connection_id> GET /user/repos # proxy: native provider API
connect proxy POST <connection_id>/some/path --data '{..}'
connect use <connection_id> # set a default for later commands
```

Tool runs (`/uts/v1/tools/../execute`) and webhook subscriptions (`connect subscribe`) also take the `connection_id`. Deep dives on those surfaces are in the `connect` skill.

## Managing the fleet

```bash
connect connections list # all your connections
connect connections list --service github # filter by service (= ?service_id=github)
connect connections show <connection_id> # service, status, scopes, tenant_id, created_at
connect connections delete <connection_id> # revoke = DELETE /connections/{id}
```

A connection's `status` (e.g. `active`) tells you whether it still works. `delete` is the revoke — it removes the stored credentials; anything acting through that `connection_id` stops working immediately.

## Errors when creating or using a connection

| Code | Cause | Fix |
| --- | --- | --- |
| `404` | Service id unknown, or `connection_id` not found | Check `connect services list`; check `connect connections list` |
| `400` | Wrong auth path for the service (api-key path on an OAuth service, or vice versa) | Use the path matching `connect services show <svc>` |
| `403` | That `connection_id` belongs to another org | List your own connections; you can't act cross-org |
| `409` | Workspace/account already linked to another connection in this org | Reuse the existing connection instead of re-creating |
| `422` | Bad request body (missing `api_key`, reserved key in `additional_config`, bad `redirect_uri`) | Fix the body; keep credential keys out of `additional_config` |
| `401` on a later call | Stored OAuth credential expired and isn't refreshable | `connect connections show <conn>`; re-do OAuth, or update the api-key connection |

## References

| File | When to read |
| --- | --- |
| [references/multi-tenant.md](references/multi-tenant.md) | One org holding many users' connections; `tenant_id` / `tenant_app_id`; the connection record's fields; the 409 uniqueness rule |

The live `GET /openapi.json` at `https://connect.service.ab0t.com/openapi.json` is the authoritative contract for request/response shapes.
