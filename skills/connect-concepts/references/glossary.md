# Glossary — every Connect mesh term

Definitions, not commands. Read top-down once and the rest of the system reads
plainly. The terms group into four families: the mesh and its tenancy, the
connection itself, the three action surfaces, and the event model.

## Table of contents

- [The mesh and tenancy](#the-mesh-and-tenancy)
 - [mesh](#mesh)
 - [org / tenant](#org--tenant)
 - [tenant_id](#tenant_id)
 - [tenant_app_id](#tenant_app_id)
 - [profile](#profile)
- [The connection](#the-connection)
 - [connection](#connection)
 - [connection_id](#connection_id)
 - [consent vs action](#consent-vs-action)
 - [scope](#scope)
- [The three surfaces](#the-three-surfaces)
 - [proxy](#proxy)
 - [UTS tool](#uts-tool)
- [The event model](#the-event-model)
 - [event](#event)
 - [subscription](#subscription)
 - [webhook](#webhook)
 - [delivery](#delivery)

---

## The mesh and tenancy

### mesh
The Connect **integration mesh** — one authenticated gateway between you and many
third-party services. You talk to the mesh; the mesh stores credentials, injects
them on outbound calls, refreshes tokens, adds provider-specific headers, applies
rate limits, audits, and enforces tenancy. It turns N bespoke integrations into
one shape. Reached at `connect.service.ab0t.com` (renaming to
`connect.service.ab0t.com`).

### org / tenant
The ownership boundary. Your **org** (tenant) owns every connection you create
and is the unit of isolation. **Tenancy is enforced server-side**: you only ever
see and act on your own org's data, and an identifier belonging to another org is
rejected. Authenticating to the mesh is what proves you are your org.

### tenant_id
The **upstream** tenant/workspace/installation identifier recorded on a
connection — *who the connection is on the provider's side*, independent of who
owns it in your org. Examples: a Slack `team_id`, a GitHub `installation_id`, an
Atlassian `cloudId`, a Discord guild id. Captured automatically for OAuth
services; suppliable for API-key services that have a tenant concept. Within one
org, a given upstream tenant can be claimed by only one connection — a second
attempt for the same `(service, tenant_id)` is rejected as a duplicate. Do not
confuse the upstream `tenant_id` with your own org/tenant boundary above.

### tenant_app_id
The upstream **app/installation** identifier *within* a tenant — a finer-grained
companion to `tenant_id`. Like `tenant_id`, it is a top-level field on the
connection record so inbound events can be routed to the right connection.

### profile
A client-side bundle of "which mesh and which org am I pointed at" — an API URL,
an auth URL, an org slug, and its own stored credentials. Profiles keep
environments (e.g. dev vs prod) isolated; switching profiles never logs you out
of the others. A profile is a *client* concept (it lives in your CLI config), not
a server-side entity.

## The connection

### connection
The central object. A connection pairs **your org + one service + stored,
encrypted credentials for one account of that service**. It is the durable
artifact of consent. The mesh never returns the raw credential — only metadata
plus an "encrypted" sentinel. One org holds many connections (many users, many
accounts, many services).

### connection_id
The opaque identifier for a connection — the **spine of the entire system**.
Everything you do downstream (proxy call, tool run, event subscription) names a
`connection_id`, and the mesh acts as that account. It is the unit of identity,
isolation (a foreign one is rejected), and audit. Hold it and you can act.

### consent vs action
The deliberate split at the heart of the model.
- **Consent** is the **one-time** act of creating a connection: a human approves
 OAuth scopes in a browser, or someone pastes an API key. It produces the
 connection.
- **Action** is everything afterward — **unlimited and headless**. An agent acts
 through the `connection_id` forever with no human in the loop and no
 re-consent.

The practical rule: never re-consent if a valid connection already exists; reuse
its `connection_id`.

### scope
A permission a provider grants to an OAuth connection (e.g. read calendars, send
messages). Scopes are fixed at consent time and recorded on the connection. If a
later operation — for instance subscribing to certain event types — needs a scope
the connection was never granted, the connection must be re-consented with the
missing scope. API-key connections don't have OAuth scopes; their capability is
whatever the static secret itself permits.

## The three surfaces

### proxy
The first surface: a **transparent passthrough** to a provider's *native* API.
You address the provider's own path and body; the mesh injects the upstream
credential and any provider-specific headers server-side, runs the call, and
returns the provider's raw response. The HTTP method on your call is the method
used upstream. The proxy is the **universal fallback** — any provider endpoint is
reachable here, even ones no tool wraps.

### UTS tool
The second surface. **UTS** = Unified Tool Schema. A tool is a **normalized**,
named `category/tool` operation with a typed input envelope, a typed output
schema, contract pre/postconditions, and a structured error catalog. Tools call
*through* the proxy under the hood (credentials still injected), but add
validation and consistent shapes across providers. Prefer a tool when one exists;
fall back to the proxy otherwise.

## The event model

The inbound direction: providers notify you when something happens upstream.

### event
A single thing that happened on a provider (a GitHub push, a Stripe invoice paid,
a Slack message) that the mesh received on your behalf. The mesh keeps an audit of
the inbound events that arrived in your scope. Events flow through a connection —
each is tied to the `connection_id` it came from.

### subscription
A standing instruction: "deliver these **event types** from **this connection**."
It is the configuration that turns raw provider events into things delivered to
you. A subscription can have a paused/active state. The **same** subscription
feeds both delivery modes — a webhook URL and the live websocket stream.

### webhook
The **push** delivery mode. The mesh POSTs each event to an HTTPS URL you host.
Requires a public inbound endpoint. Deliveries are **at-least-once**, retried with
backoff, and replayable; each POST is **HMAC-signed**, so you verify the signature
on the raw body before trusting it. Best for durable server-to-server pipelines.
(The alternative mode is a **websocket** stream — you dial out a live socket, no
public URL, best-effort while connected, best for live UIs and agents.)

### delivery
One **outbound POST attempt** of an event to your webhook URL. Distinct from the
event itself: the *event* is what the provider sent in; the *delivery* is the
mesh's attempt to hand it to your receiver. A delivery has attempts and a status,
and a failed/timed-out delivery can be retried. (Websockets have no deliveries —
they stream events directly, without the per-POST retry machinery.)
