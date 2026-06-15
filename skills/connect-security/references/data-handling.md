# Data handling — what's stored, returned, logged, and how to revoke

The security-questionnaire answers for the ab0t Connect mesh. Pairs with [signature-verification.md](signature-verification.md) (inbound delivery authenticity).

> Host: `connect.service.ab0t.com`; CLI: `connect`.

## What is stored

| Stored | Form |
| --- | --- |
| Upstream credentials — OAuth access & refresh tokens, API keys, client secrets, webhook signing secrets | **Encrypted at rest (AES‑256 / Fernet).** Decrypted only in-process at call time |
| Connection metadata — service id, status, granted scopes, tenant metadata, timestamps | Plain (non-secret) |
| Inbound events you received, and outbound delivery records | Scoped to your org; an audit trail of what arrived and what was sent to your URL |

The secret material is the only sensitive part, and it is always encrypted. Connection metadata carries no secret.

## What is returned to callers

- **Never a secret.** A connection's credential field returns the sentinel `{"status":"encrypted"}` — never the token or key. You cannot read back what you stored.
- Connection responses return **metadata only**: service, status, scopes, tenant info, `created_at`.
- A webhook signing secret is shown **once** at subscribe time (and cached locally per profile / retrievable with `--secret`), because *you* need it to verify deliveries — it is not part of normal connection reads.

## What is logged vs. not

- **Not logged:** plaintext secrets. Encryption/decryption failures are logged as error type only, never the secret value or the credential payload.
- **Logged / observable to you:** inbound events (the audit of what providers sent) and outbound deliveries with per-attempt status — all scoped to your org, useful for debugging and compliance.

```bash
connect webhooks events list # raw inbound events you received (your scope only)
connect webhooks deliveries list # outbound POSTs to your URL + status
connect webhooks deliveries show <id> --attempts
```

## Scopes & least privilege

- An OAuth connection holds **only** the scopes its one-time consent granted. Request the minimum the task needs.
- The mesh will not silently over-reach: an operation needing a scope the connection lacks is **refused** (e.g. creating a subscription whose `event_types` need un-granted scopes returns `422 missing_scopes`).
- To widen access, re-consent the connection with the additional scopes — a deliberate, human-approved step, not an automatic escalation.
- The static-secret (api-key) connection path **rejects** reserved credential/routing keys (`access_token`, `refresh_token`, `tenant_id`, …) in `additional_config` (`422`); you can't inject OAuth tokens through it.

## Token lifecycle

- Upstream OAuth access tokens are **refreshed/rotated server-side** by the mesh using the stored refresh token. You never handle upstream tokens.
- Providers that issue a **new refresh token on every refresh** (rotating-refresh) are handled — the mesh replaces the stored refresh token transparently.
- If a stored credential can no longer be refreshed, calls through it return `401`; re-do the OAuth consent (or update the api-key connection) to restore it.
- Mesh API keys (`ab0t_sk_…`) don't expire on a timer; rotate them in the mesh and re-export. JWT user tokens auto-refresh until the refresh itself expires.

## Revocation

Deleting a connection removes the encrypted credentials immediately; anything acting through that `connection_id` stops working at once.

```bash
connect connections delete <connection_id>
```

## Tenant isolation

- Connections, subscriptions, events, and deliveries are **org-scoped** server-side; you see and act on only your own org's data.
- A resource (e.g. a `connection_id`) belonging to another org is **denied with `403`** — there is no cross-tenant read or action, and no query parameter that crosses the boundary.
- The realtime stream is scoped to your token's org; you receive only events in your scope.

## Transport

- All API traffic is **HTTPS (TLS)**; the realtime stream is **`wss://`**; webhook receiver URLs are HTTPS.
- Secrets never cross the wire in plaintext.

## One-line trust summary

Your secrets are encrypted at rest and never handed back; the mesh acts only through connections you consented to, within their granted scopes; tokens are refreshed for you; every inbound delivery is signed for you to verify; and your org's data is walled off from every other tenant.
