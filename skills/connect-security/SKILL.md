---
name: connect-security
description: The security and data-handling model of the ab0t Connect mesh — what an enterprise client needs to trust the system. Use when someone asks "is my data secure", "how are credentials stored", "do you store my tokens", "encryption at rest", "what's the security model", "how does data handling work", "what is the consent model", "what can the mesh access", "what can't the mesh see", "how do I verify a webhook signature", "HMAC verification", "is this delivery authentic", "how does token rotation / refresh work", "do tokens expire", "scopes and least privilege", "tenant isolation", "can another org see my connections", "cross-tenant access", "is it encrypted in transit", "TLS / wss", or asks compliance / trust / due-diligence questions about credentials, consent, scopes, webhooks, token lifecycle, transport, or tenant isolation. Read this to answer a security questionnaire or reassure an enterprise buyer; for the connection lifecycle itself see connect-connections, for receiving events see connect-realtime, for CLI auth see connect-auth.
---

# Connect mesh — security & data-handling

What an enterprise client needs to trust the ab0t Connect mesh: how credentials are stored, what consent the mesh requires, what it can and can't reach, how to verify inbound events, how tokens are kept fresh, and how your tenant is isolated from everyone else's.

> The mesh CLI is `connect`. Public host: `connect.service.ab0t.com` (dev: `connect.dev.ab0t.com`); realtime is `wss://`.

## The seven properties

1. **Credentials are encrypted at rest and never returned.** Stored secrets (OAuth access/refresh tokens, API keys, client secrets, signing secrets) are encrypted at rest with AES‑256 (Fernet) and decrypted only server‑side at the moment a call is made on your behalf. API responses **never** return a secret — a connection's credential field reads `{"status":"encrypted"}`, a sentinel, not the value. You cannot read back what you stored; neither can anyone else.
2. **Consent is explicit and one-time.** An OAuth connection exists only after a human approves the requested scopes in the provider's own consent screen. There is no way around that approval. Creating the connection is the consent; acting through its `connection_id` afterward is headless. The mesh acts **only** through connections you created.
3. **Least privilege.** Request only the scopes the work needs. A connection is limited to the scopes its consent granted — if a later operation needs a scope you never requested, it is refused (e.g. a subscription returns `422 missing_scopes`) rather than silently over-reaching. Narrow the grant; re-consent only when you genuinely need more.
4. **Inbound deliveries are signed — always verify.** Every webhook the mesh POSTs to your URL carries an HMAC‑SHA256 signature over the raw body. **Verify it before you trust the delivery.** This proves the event came from the mesh and was not tampered with or replayed. See [references/signature-verification.md](references/signature-verification.md).
5. **Token lifecycle is managed server-side.** OAuth access tokens are refreshed/rotated automatically by the mesh using the stored refresh token — including providers that issue a **new refresh token on every refresh** (rotating-refresh). You never handle, store, or rotate upstream provider tokens; you only hold an opaque `connection_id`.
6. **Everything is encrypted in transit.** All API traffic is HTTPS (TLS); the realtime stream is `wss://`. Webhook receiver URLs are HTTPS. Secrets never cross the wire in plaintext.
7. **Your tenant is isolated.** Connections, subscriptions, events, and deliveries are scoped to your org server-side. You see and act on **only** your own org's data. A `connection_id` (or any resource) belonging to another org is denied with `403` — there is no cross-tenant read or action.

## Credential storage & the encrypted sentinel

A connection pairs **your org** + **one service** + **encrypted credentials for one account**. The secret material is encrypted before it is written and is only ever decrypted in-process to make the upstream call. It is never logged, never echoed, and never serialized into a response.

```bash
connect connections show <connection_id>
# returns service, status, scopes, created_at, tenant metadata..
# the credential reads: {"status":"encrypted"} <- sentinel, never the secret
```

The api-key connection path additionally **rejects credential/routing keys** (`access_token`, `refresh_token`, `tenant_id`, …) smuggled in via `additional_config` (`422`) — you cannot inject OAuth tokens through the static-secret endpoint.

**Revocation is immediate.** Deleting a connection removes the stored credentials; anything acting through that `connection_id` stops working at once.

```bash
connect connections delete <connection_id> # revoke = remove credentials now
```

## Consent & what the mesh can / can't access

| The mesh CAN | The mesh CANNOT |
| --- | --- |
| Act through connections **you** created, within **their granted scopes** | Reach a provider account you never connected |
| Use a connection's scopes to call that one provider on your behalf | Exceed the scopes the consent granted (refused, e.g. `422 missing_scopes`) |
| Refresh that connection's upstream token server-side | Touch another org's connections, events, or deliveries (`403`) |
| Receive events you subscribed to, scoped to your org | Return any stored secret to any caller (sentinel only) |

Consent is collected once per OAuth connection in the **provider's own** approval screen. The mesh never sees the user's provider password — only the post-consent tokens, which it then encrypts.

## Authentication & tenant isolation

Every authenticated request carries `Authorization: Bearer <token>` — either a JWT user token or an `ab0t_sk_…` mesh API key (the mesh also accepts `X-API-Key`). Each token is bound to an org; the mesh derives your tenancy from it and scopes every read and write to that org. The realtime socket authenticates on connect (`?token=…`); a missing token closes with `4001`, an invalid/out-of-scope token with `4003`, and you only ever receive events in your own scope.

There is no parameter or path that lets one org reach another's data. Cross-tenant attempts are denied with `403`, not filtered silently.

## Verifying inbound webhooks (the one thing you implement)

The single security step on your side: **verify the HMAC signature on every delivery before trusting it.** The signing secret is shown once when you subscribe — store it. The signature is computed over the **raw request body only** (not Stripe-style `t.body`); verify the exact bytes you received, in constant time, and reject deliveries outside a small timestamp window to guard against replay.

```python
import hmac, hashlib, time

def verify(raw_body: bytes, sig_header: str, secret: str, max_skew=300) -> bool:
 parts = dict(p.split("=", 1) for p in sig_header.split(",") if "=" in p)
 t, v1 = parts.get("t"), parts.get("v1")
 if not t or not v1:
 return False
 if abs(time.time() - int(t)) > max_skew: # replay guard
 return False
 expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
 return hmac.compare_digest(expected, v1) # sign the BODY, nothing else
```

Full pattern in Python / JS / Go, plus the header format and replay rules: [references/signature-verification.md](references/signature-verification.md).

## References

| File | When to read |
| --- | --- |
| [references/signature-verification.md](references/signature-verification.md) | Verify an inbound webhook delivery — header format, the HMAC-over-raw-body rule, Python/JS/Go, replay window, constant-time compare |
| [references/data-handling.md](references/data-handling.md) | What is stored, what is returned, what is logged vs. not, scopes & least-privilege, revocation, transport, retention — the security-questionnaire answers |
