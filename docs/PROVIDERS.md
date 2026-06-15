# Provider catalog

The services you can connect through the ab0t integration mesh with `connect`.
Connect once (OAuth consent, or a single API-key paste), then act through the
connection's `connection_id` via the proxy, the normalized tools (UTS), or
webhooks — credentials injected and tenancy enforced server-side.

This page is the human-readable catalog. For the live, authoritative list and
per-service detail use:

```bash
connect services list # the connectable catalog
connect services show <service_id> # auth type, scopes, webhook support
connect services events <service_id> # the authoritative webhook event types
```

## How to read the table

- **Auth** — how a connection is born:
 - **OAuth** = one-time human consent in a browser (per account).
 - **API key** = paste a static secret once, fully headless.
- **Proxy** — call the provider's native API path through `connect call` / `proxy`.
- **Tools** — normalized `category/tool/execute` operations (where coverage exists);
 discover with `connect tools catalog`.
- **Webhooks** — inbound provider events delivered to your URL (where the
 provider supports them); confirm with `connect services events <svc>`.

## Catalog

| Service | Auth | What you can do | Webhooks |
| --- | --- | --- | --- |
| **Slack** | OAuth | Post and read messages, manage channels, react, look up users — via proxy (`/chat.postMessage`, …) or messaging tools | Slack `event_callback` events |
| **GitHub** | OAuth | Repos, issues, pull requests, commits, org/user data — proxy (`/user/repos`, `/repos/{o}/{r}/issues`) or tools | `push`, PR/issue events |
| **Discord** | OAuth | Channels, messages, guild operations | Gateway/interaction events |
| **Jira** | OAuth | Issues, projects, transitions, comments, search (JQL) | Issue/workflow events |
| **Stripe** | API key | Payments, customers, invoices, subscriptions, refunds (read/write per key scope) | Payment/invoice/subscription events |
| **Google — Calendar** | OAuth | List/create/update events and calendars (`/calendar/v3/..`); calendar tools | Push/notification channels |
| **Google — Sheets** | OAuth | Read/write ranges, create/format sheets | — |
| **Google — Slides** | OAuth | Create/update presentations and slides | — |
| **Google — Gmail** | OAuth | Read/send mail, manage labels and threads | Push (Pub/Sub) |
| **Google — Drive** | OAuth | List/upload/download/share files and folders | Change notifications |
| **Microsoft** | OAuth | Microsoft Graph surface — mail, calendar, files, Teams (per granted scopes) | Graph change notifications |
| **Salesforce** | OAuth | Records (CRUD), SOQL queries, sObjects, bulk operations | Platform/streaming events |
| **SendGrid** | API key | Send transactional/marketing email, manage templates and contacts | Event webhook (delivery/opens/bounces) |
| **Twilio** | API key | Send SMS/MMS, place calls, manage numbers and messaging services | Status callbacks |
| **Xero** | OAuth | Accounting: invoices, contacts, payments, bank transactions, chart of accounts | Accounting events |
| **Meta Ads** | OAuth | Campaigns, ad sets, ads, audiences, insights/reporting across Meta ad accounts | Ad-account events |
| **Google Ads** | OAuth | Campaigns, ad groups, keywords, budgets, performance reporting (GAQL) | Account/change events |

> The exact set of services, scopes, tools, and supported webhook event types
> evolves. `connect services list` and `GET /openapi.json` are the source of
> truth; treat this table as an orientation map.

## Picking proxy vs. tools

- Reach for **tools** first when a tool exists for the operation: the input is
 schema-validated, errors are structured, and connection ownership is checked
 for you. Discover with `connect tools catalog` / `connect tools show
 <cat>/<tool> --examples`.
- Fall back to the **proxy** (`connect call <conn> <METHOD> <path>`) for any
 provider operation not yet covered by a tool — it's a transparent passthrough
 to the provider's own API with the credential injected.

## Connecting examples

```bash
# OAuth (one-time browser consent), then act headless forever:
connect add xero
connect add google_calendar
connect add github

# API-key services connect headlessly:
connect add sendgrid --api-key SG.xxxx --name prod
connect add stripe --api-key sk_live_xxxx --name billing

# Then list and act:
connect connections list
connect call <connection_id> GET /user/repos
connect tools show google-calendar/create_event --examples
```

See [`USAGE.md`](USAGE.md) for the full connection model (consent vs. action)
and the proxy / tools / webhook surfaces.
