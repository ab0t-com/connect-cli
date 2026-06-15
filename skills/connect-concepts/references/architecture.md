# Architecture — the request lifecycle, conceptually

What actually happens between you and a provider when you act through the Connect
mesh. This is the conceptual picture, not an endpoint reference.

## The shape of every interaction

You never talk to the provider directly. You talk to the mesh, addressing **one
account** by its `connection_id`, and the mesh does the provider-facing work.

```
 you ──► the mesh ──► the provider ──► the mesh ──► you
 │ (1) identify the org + connection
 │ (2) inject the upstream credential
 │ (3) add provider-specific headers / refresh tokens
 │ (4) rate-limit + audit
 └──────────────────────────────────────┘
 you send NO provider auth; you get the provider's response back
```

## The outbound lifecycle (proxy and tools)

When you make a call through a connection, the mesh does roughly the following
between receiving your request and returning a response:

1. **Authenticate you to the mesh.** Your request carries *your* mesh credential
 (a user token or an API key) — never a provider token. This proves which org
 you are.

2. **Resolve the connection and enforce tenancy.** The `connection_id` you named
 is looked up. If it doesn't belong to your org, the call is rejected before
 anything reaches the provider — the org boundary is enforced server-side, not
 trusted from the client.

3. **Inject the upstream credential.** The mesh swaps in the stored, encrypted
 credential for that connection's account — replacing/stripping any auth header
 you might have sent. You never hold or transmit the provider's secret.

4. **Refresh if needed.** If the connection's OAuth access token has expired, the
 mesh refreshes it transparently before making the call. This is invisible to
 you and is why a long-lived connection keeps working without re-consent.

5. **Add provider-specific plumbing.** Some providers need extra headers, tenant
 identifiers, or signed proofs on every request. The mesh computes and attaches
 these for you, so your call only ever contains the logical request.

6. **Rate-limit and audit.** The call is mediated against rate limits and
 recorded for audit, correlated by a request id.

7. **Call the provider and pass the response back.** For the **proxy**, you get
 the provider's raw response (its status, headers, body) passed through. For a
 **UTS tool**, the same passthrough happens underneath, but the result is shaped
 into the tool's typed output and any failure is mapped to a structured error
 from the tool's catalog.

The difference between the two outbound surfaces is only altitude: the proxy is
the raw passthrough; a UTS tool wraps that passthrough with validation in front
and a normalized shape behind. Both ride the same auth-injection / refresh /
tenancy / audit machinery.

## The inbound lifecycle (events)

Events run the same machinery in reverse — a provider initiates, and the mesh
delivers to you:

1. **A provider fires an event** (a push, a payment, a message).

2. **The mesh receives and routes it.** It resolves the event to the right
 connection (this is what the upstream `tenant_id` / `tenant_app_id` on the
 connection record are for) and records it in your inbound-event audit.

3. **It matches the event to your subscriptions** — the standing "deliver these
 event types from this connection" instructions.

4. **It delivers, by the chosen mode:**
 - **Webhook (push):** the mesh POSTs the event to your hosted URL, signs it so
 you can verify authenticity, and retries with backoff until you acknowledge —
 keeping a per-delivery record you can inspect and replay.
 - **Websocket (stream):** the mesh pushes the event down a live socket you
 opened, scoped to your org, with no retry/replay — best-effort while
 connected.

The same subscription can feed both modes at once: webhooks as the durable system
of record, a websocket for live updates.

## Why this shape

Every property that makes the mesh useful comes from the mesh sitting *between*
you and the provider and owning the credential:

- **You never hold provider secrets** — they live encrypted in the mesh and are
 injected per call.
- **Tokens refresh themselves** — long-lived connections keep working.
- **One shape for N providers** — the per-provider plumbing is absorbed by the
 mesh, so your calls stay logical and uniform.
- **Isolation and audit are server-side** — not features you have to build, and
 not things a client can bypass.

This is the whole value proposition restated as architecture: the mesh does the
credential-handling, refresh, provider-plumbing, rate-limiting, and audit so that
you (or an agent) only ever express the logical request and act through a
`connection_id`.
