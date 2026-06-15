---
name: connect-concepts
description: >-
 The mental model of the ab0t Connect integration mesh — read this FIRST to
 understand the whole system before touching any how-to skill. Use when someone
 asks "how does the connect mesh work", "what is the connect mesh", "explain the
 system", "mental model", "concepts", "architecture overview", "getting started
 understanding", "the big picture", or asks what a "connection / connection_id /
 org / tenant / tenant_id / proxy / UTS tool / webhook / subscription / delivery
 / event / scope / mesh" IS (definition, not commands). Use when deciding between
 "the three surfaces" (proxy vs UTS tools vs events), grasping "consent vs
 action", understanding the connection lifecycle, or choosing "push vs stream"
 (webhooks vs websockets) conceptually. This skill CONCEPTUALIZES and defines —
 for step-by-step commands use the connect-connections, connect-proxy,
 connect-tools, connect-realtime, or connect-auth skills instead.
---

# ab0t Connect — the mental model

> The CLI is `connect`; the host is
> `connect.service.ab0t.com`.

This is the read-this-first overview. It explains *what the system is* and *how
the pieces fit*. It does not teach commands — each how-to lives in its own skill.

## What the mesh is

The Connect mesh is one authenticated gateway that sits between you and many
third-party services (Slack, GitHub, Google, Stripe, Xero, Meta Ads, …). You
talk to the mesh; the mesh talks to the providers. In return it handles the
parts of every integration that are otherwise repeated, fiddly, and
security-sensitive:

- **Auth** — it stores each provider's credential encrypted and injects it on
 every outbound call. You never hold or send a provider token.
- **Token refresh** — expired OAuth tokens are refreshed transparently.
- **Provider-specific plumbing** — required headers and proofs are added for you.
- **Rate limiting and audit** — every call is mediated and recorded.
- **Tenancy** — your data is isolated server-side; you only ever see your own.

So instead of N bespoke integrations (each with its own auth, refresh, headers,
and webhook receiver), you get **one shape** for all of them.

```
 you ──HTTPS──► the Connect mesh ──► Slack / GitHub / Google /
 (proxy · tools · Stripe / Xero / Meta Ads / …
 events)
 │
 └─ encrypted credentials, one per connected account
```

## The org → connection → act model

Three layers, top to bottom:

1. **Your org** owns everything. Authenticating to the mesh proves you are your
 org. Tenancy is enforced server-side: another org's data is invisible to you,
 and acting on it is impossible (a foreign id is rejected).
2. **Connections** live under your org. A **connection** pairs your org + one
 service + stored, encrypted credentials for **one account** of that service.
 One org holds **many** connections — many users, many accounts, many services
 side by side.
3. **You act through a connection.** Each connection has an opaque
 `connection_id`. Every single thing you do downstream — a proxy call, a tool
 run, a webhook subscription — names a `connection_id`, and the mesh acts as
 that account.

The `connection_id` is the spine of the whole system. Hold it, and you can act;
it is the unit of identity, isolation, and audit.

## Consent vs action — the key idea

Creating a connection and using a connection are deliberately split:

- **Consent** happens **once**, when the connection is born. For OAuth services a
 human approves scopes in a browser; for API-key services someone pastes a
 static secret. Either way it produces a durable consent artifact: the
 connection.
- **Action** happens **forever after**, headless. Once the connection exists, an
 agent acts through its `connection_id` with no human in the loop and no
 re-consent — unlimited proxy calls, tool runs, and event subscriptions.

This is what makes the mesh agent-friendly: the only step that may need a human
is the one-time consent, and only for OAuth providers. Everything after is
automatable. The rule that falls out of this: **never re-consent if a valid
connection already exists** — list first and reuse.

## The three surfaces — and when to use each

Once you hold a `connection_id`, there are three ways to act through it. They are
not alternatives to learn one of — they are three tools for three jobs.

```
 ┌──────────────► PROXY — the provider's own native API
 connection_id ───────┤
 ├──────────────► UTS TOOLS — normalized, validated operations
 │
 └──────────────► EVENTS — provider events flowing back to you
```

| Surface | What it is | Reach for it when |
|---|---|---|
| **Proxy** | A transparent passthrough to the provider's *native* API path. You write the provider's own path and body; the mesh injects auth and returns the raw response. | You know the provider's API, or need an endpoint that no tool wraps. The proxy is the **universal fallback** — full provider coverage. |
| **UTS tools** | A **normalized** layer: named `category/tool` operations with typed input/output schemas, contract checks, and structured errors. | You want one consistent, validated shape across providers, and a tool exists for the operation. **Prefer a tool when one exists.** |
| **Events** | Provider events flowing **back** to you (the inbound direction). | A provider should *notify you* when something happens upstream. |

Proxy and tools are the same outbound action at two altitudes: the proxy is raw
and universal, tools are normalized and safe. The rule of thumb: **prefer a UTS
tool when one exists; fall back to the proxy for anything it doesn't cover.**

## The event model — push vs stream

Events (surface 3) are the inbound direction: a provider fires, and the mesh
delivers it to you. There are two delivery modes — pick by what you are building.

| | **Webhooks** (push) | **Websockets** (stream) |
|---|---|---|
| How | The mesh POSTs each event to a URL you host | You open a live socket and events arrive on it |
| Needs a public URL? | **Yes** — an inbound HTTPS endpoint | **No** — you dial out (works behind NAT) |
| Durability | At-least-once, retried with backoff, replayable | Best-effort while connected; no retry/replay |
| Verify? | You verify an HMAC signature on every POST | Auth is on the socket; no per-message signing |
| Best for | Server-to-server pipelines, durable processing, audit | Live UIs, dashboards, agents, browsers, anything ephemeral |

Both modes start from the same **subscription** ("deliver these event types from
this connection"). One subscription can feed a webhook URL and a websocket stream
at once. Many apps use **both**: webhooks as the durable system of record, a
websocket to push live updates into a UI. Choose webhooks when you must not lose
an event; choose websockets when you want events live in a client that can't
expose an inbound URL.

## How it all fits

The whole system reads as one sentence:

> **Your org** creates a **connection** (one-time **consent**), which yields a
> **`connection_id`**, which you then use — **headlessly, forever** — to **act**
> on three surfaces: **proxy** the provider's API, run normalized **UTS tools**,
> or receive **events** back (as **push** webhooks or a **stream** websocket).

The lifecycle:

```
 AUTH to the mesh ──► CONNECT a service ──► hold a connection_id ──► ACT
 (prove your org) (one-time consent) (durable, reusable) │
 ├─ proxy a native call
 ├─ run a UTS tool
 └─ receive events (push/stream)
```

Authenticating to the mesh and having connections are **orthogonal** — you can be
authenticated with zero connections. Auth proves your org; a connection grants
access to a provider account. You need both to act.

## Where to go next

| You want to… | Skill |
|---|---|
| Understand or set up a **connection** (OAuth vs API key, `connection_id`, multi-tenant) | `connect-connections` |
| Call a provider's **native API** through the passthrough | `connect-proxy` |
| Run **normalized, validated tools** (UTS) | `connect-tools` |
| **Receive events** — webhooks or websockets | `connect-realtime` |
| **Authenticate the CLI** / manage profiles, keys, environments | `connect-auth` |
| Drive everything from the **CLI** end to end | `connect` |

## References

| File | When to read |
|---|---|
| [references/glossary.md](references/glossary.md) | Definition of every term — connection, `connection_id`, org/tenant, `tenant_id`/`tenant_app_id`, consent vs action, proxy, UTS tool, webhook, subscription, delivery, event, profile, scope, mesh |
| [references/architecture.md](references/architecture.md) | The request lifecycle conceptually — what the mesh does between you and the provider (auth injection, refresh, rate-limit, audit) |
