# Multi-tenant connections

A connection is **org-scoped**, not user-scoped. One org can hold many connections at once, and the mesh is built so a single org can fan out across many upstream accounts and users. This is what an app builder needs when their product connects services on behalf of *their* users.

## Why an org holds many connections

- Many **users** in your org can each connect their own account — one `connection_id` per user-account-service.
- Many **accounts** of the same service can coexist (e.g. a prod and a staging Stripe; two different Slack workspaces).
- Many **services** are connected side by side (a user's Google + GitHub + Slack).

You list and filter them all from one place:

```bash
connect connections list # everything your org owns
connect connections list --service slack # = GET /connections/?service_id=slack
```

Tenancy is enforced **server-side**. A `connection_id` that belongs to another org returns `403` no matter what you do with it — there is no way to act across the org boundary.

## tenant_id and tenant_app_id — tracking the upstream identity

A connection optionally records **who it is upstream**, independent of who owns it in your org:

- **`tenant_id`** — the upstream tenant/workspace/installation identifier. Examples: a Slack `team_id` (`T0A0H1AB797`), a GitHub `installation_id`, an Atlassian `cloudId`, a Discord guild snowflake.
- **`tenant_app_id`** — the upstream app/installation identifier within that tenant.

For OAuth services these are captured automatically during the callback (extracted from the token exchange). For API-key services that still have a tenant concept, you may supply them when you create the connection:

```bash
curl -X POST "$URL/connections/<svc>/api-key" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"api_key":"..","name":"prod","tenant_id":"T0A0H1AB797"}'
```

Charset for these identifiers is alphanumerics plus `- _ . :` (covers known provider formats). They are top-level fields on the connection record — not buried in credentials — so inbound webhook/event routing can resolve an upstream event to the right connection without scanning. They are **reserved keys**: you cannot set them via `additional_config` (that returns `422`).

## The uniqueness rule — 409 on a claimed workspace

Within an org, an upstream tenant can be **claimed by only one connection**. If you try to create a second connection for a `(service, tenant_id)` that's already linked, you get:

```
409 Workspace already linked to another connection (Reference: <id>)
```

This is by design — it keeps one authoritative connection per upstream workspace and prevents duplicate/fabricated claims. The fix is to **reuse the existing connection** rather than create a new one: list, find it, act through its `connection_id`. The error never echoes the tenant_id back.

## What a connection record exposes

`connect connections show <connection_id>` (= `GET /connections/{id}`) returns metadata only — credentials are always `{"status":"encrypted"}`. The fields you can rely on:

| Field | Meaning |
| --- | --- |
| `connection_id` | The opaque id you act through |
| `service_id` | Which service this connects |
| `connection_name` | Your label for it |
| `status` | Whether it currently works (e.g. `active`) |
| `scopes` | Granted scopes (for OAuth) |
| `tenant_id` / `tenant_app_id` | Upstream identity, if any |
| `account_info` | Upstream account details captured at connect time |
| `created_at` / `last_used` / `last_updated` / `expires_at` | Lifecycle timestamps |

`credentials` is always the encrypted-status sentinel — the raw secret is never returned by the API.
