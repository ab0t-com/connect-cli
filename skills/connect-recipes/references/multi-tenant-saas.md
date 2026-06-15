# Recipe 5 — Multi-tenant SaaS: each customer connects their own accounts

The big commercial pattern. You're building a product; each of YOUR customers
connects THEIR OWN third-party accounts (their Slack, their Xero, their GitHub),
and your app acts on each customer's behalf through that customer's
`connection_id` — with tenant isolation enforced by the mesh, not by you.

> CLI `connect`. Host
> `connect.service.ab0t.com`. `URL` = host,
> `KEY` = an `ab0t_sk_live_…` mesh key.

## Contents

- [The model in one breath](#the-model-in-one-breath)
- [Pattern A — one ab0t org per customer](#pattern-a--one-ab0t-org-per-customer-strong-isolation)
- [Pattern B — one org, many users](#pattern-b--one-org-many-users-per-user-connections-inside-your-org)
- [Tracking upstream identity — tenant_id / tenant_app_id](#tracking-the-upstream-identity--tenant_id--tenant_app_id)
- [Headless / service-account connections](#headless--service-account-connections-no-per-customer-browser)
- [What a connection record exposes](#what-a-connection-record-exposes-for-your-product-ui)
- [End-to-end: "let my users connect their accounts"](#end-to-end-let-my-users-connect-their-accounts)

## The model in one breath

- A **connection** is **org-scoped**: your org owns it, it pairs one service with
 encrypted credentials for one account of that service, and it's addressed by an
 opaque `connection_id`.
- One org holds **many** connections — many users, many accounts, many services,
 side by side.
- **Everything you do acts through a `connection_id`.** Pick the right one and
 you're acting as that customer's account; the mesh injects its credential.
- **Tenant isolation is server-side.** A `connection_id` that belongs to another
 org returns `403` no matter what — there is no cross-org action path.

## Pattern A — one ab0t org per customer (strong isolation)

Map each customer to their own ab0t org (each with its own mesh API key). A
customer's connections live entirely inside their org; your code selects the
customer by selecting their key, and the `403` boundary does the rest.

```bash
# Provision per customer: mint an ab0t_sk_live_… key for customer-acme's org.
# Then everything that customer connects is isolated to that key's org.
export CONNECT_API_KEY=ab0t_sk_live_<acme>
connect add slack # acme's user consents once
connect connections list # ONLY acme's connections are ever visible here
```

In your backend, hold a key per customer and act with it:

```python
def post_for_customer(customer_id, channel, text):
 key = customer_keys[customer_id] # your secret store
 requests.post(f"{URL}/uts/v1/tools/slack/post-message/execute",
 headers={"X-API-Key": key},
 json={"connection_id": conn_of(customer_id, "slack"),
 "input": {"body": {"channel": channel, "text": text}}})
```

Cross-tenant leakage is structurally impossible: customer A's key can't see or
act on customer B's `connection_id` (`403`).

## Pattern B — one org, many users (per-user connections inside your org)

Keep one ab0t org and let many of your end-users each connect their own account.
The org holds one `connection_id` per user-account-service. You select the right
connection per request from your own mapping.

```bash
connect connections list # everything your org owns
connect connections list --service slack # = GET /connections/?service_id=slack
```

Your app stores `your_user_id → connection_id` and looks it up per request:

```python
conn = lookup_connection(your_user_id, service="xero") # YOUR mapping table
requests.get(f"{URL}/proxy/{conn}/api.xro/2.0/Invoices",
 headers={"X-API-Key": ORG_KEY})
```

Because the mesh enforces ownership at the org level, your own application layer
is responsible for never handing user A the `connection_id` you stored for user
B. Keep the mapping authoritative and server-side. Pattern A is stronger when you
want the mesh itself to enforce per-customer boundaries.

## Tracking the upstream identity — `tenant_id` / `tenant_app_id`

A connection optionally records who it is **upstream**, independent of who owns
it in your org — so inbound events route to the right connection without
scanning:

- **`tenant_id`** — the upstream tenant/workspace/installation id (a Slack
 `team_id`, a GitHub `installation_id`, an Atlassian `cloudId`, …).
- **`tenant_app_id`** — the upstream app/installation id within that tenant.

For OAuth services these are captured automatically at consent. For API-key
services with a tenant concept, supply them at create time:

```bash
curl -X POST "$URL/connections/<svc>/api-key" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"api_key":"..","name":"acme-prod","tenant_id":"T0A0H1AB797"}'
```

These are reserved top-level fields — you **cannot** set them via
`additional_config` (that returns `422`). Charset is alphanumerics plus
`- _ . :`.

### The uniqueness rule — `409` on a claimed workspace

Within an org, an upstream tenant can be claimed by **only one** connection. A
second connection for the same `(service, tenant_id)` returns:

```
409 Workspace already linked to another connection (Reference: <id>)
```

This keeps one authoritative connection per upstream workspace. The fix is to
**reuse** the existing connection, not re-create it: list, find it, act through
its `connection_id`.

## Headless / service-account connections (no per-customer browser)

Two ways a connection is born — choose per service:

- **API-key services** (Stripe, SendGrid, Twilio, …): your backend holds the
 customer's secret and self-connects, fully headless. Ideal for a service
 account a customer pastes once.

 ```bash
 curl -X POST "$URL/connections/stripe/api-key" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"api_key":"sk_live_<customer>","name":"acme-billing"}'
 ```

- **OAuth services** (Slack, GitHub, Google, Xero, …): one human consent per
 customer account — unavoidable, but collected once. In an app/agent loop, emit
 the `authorization_url`, the customer approves, you resume on the new
 `connection_id`:

 ```bash
 curl -X POST "$URL/connections/slack/oauth/authorize" -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"redirect_uri":"https://connect.service.ab0t.com/connections/slack/oauth/callback"}'
 # -> { "authorization_url": "https://slack.com/oauth/..", "state":"..", "expires_in":300 }
 # customer approves -> callback exchanges tokens -> connection appears
 ```

The api-key endpoint **strips** reserved keys (`access_token`, `refresh_token`,
`tenant_id`, …) — you cannot smuggle OAuth tokens through it. Consent is consent.

## What a connection record exposes (for your product UI)

`connect connections show <connection_id>` (= `GET /connections/{id}`) returns
metadata only — `credentials` is always `{"status":"encrypted"}`:

| Field | Use in your product |
| --- | --- |
| `connection_id` | The id you act through (store it against your user/customer) |
| `service_id` | Which service this is |
| `connection_name` | Your label (e.g. `acme-billing`) |
| `status` | Show "connected / needs reconnect" in your UI |
| `scopes` | What the customer granted |
| `tenant_id` / `tenant_app_id` | Upstream identity for event routing |
| `account_info` | Upstream account details captured at connect |
| `created_at` / `last_used` / `expires_at` | Lifecycle for a connections dashboard |

## End-to-end: "let my users connect their accounts"

1. **Provision** — Pattern A (org + key per customer) for hard isolation, or
 Pattern B (one org, per-user `connection_id` mapping) for simplicity.
2. **Connect** — render a "Connect Slack" button that kicks off the OAuth
 authorize flow (or accept an API key for api-key services). Capture the
 resulting `connection_id` against your user.
3. **Act** — on every request, look up that user's `connection_id` and call a
 tool / proxy / subscribe through it. The mesh injects the credential and
 enforces the boundary.
4. **Manage** — surface `connections list` / `show` in your settings UI;
 `connections delete <id>` is the customer's "disconnect".

**Never re-consent if an `active` connection exists** — list first, reuse the
`connection_id`. That single rule keeps a multi-tenant product headless after the
first connect per customer.
