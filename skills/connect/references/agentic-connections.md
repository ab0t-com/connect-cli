# Headless & agentic connections

The central rule: **separate CONSENT (one-time, identity-granting) from ACTION (repeated, headless).** A `connection` is the durable consent artifact; once it exists, agents act through `connection_id` forever with no human in the loop. So "can an agent connect headlessly?" depends only on how that one connection is born.

## Auth to the service itself (always headless)
Agents authenticate to the connect mesh with an **API key** (`X-API-Key: ab0t_sk_live_..`) or a JWT. The API key is minted in the auth mesh for a user/org and never needs a browser. This is orthogonal to connecting third parties.

## Creating connections

### 1. API-key services — fully headless, today
SendGrid, Stripe, Twilio (anything whose auth is a static secret):
```bash
curl -X POST $URL/connections/sendgrid/api-key -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' -d '{"api_key":"SG.xxxx","name":"prod"}'
# -> { "connection_id": "conn_sendgrid_..", .. }
```
An agent that holds the secret can create the connection with zero human interaction.

### 2. OAuth services — consent is required (Google/Slack/GitHub/…)
OAuth for **user data** (Gmail, Calendar, a user's repos) requires that user to approve scopes. There is no way around the approval itself; the question is only *how* the approval is collected and *who* does it.

Standard flow:
```bash
# 1. agent asks the service for an authorization URL
curl -X POST $URL/connections/google_calendar/oauth/authorize -H "X-API-Key: $KEY" \
 -H 'Content-Type: application/json' \
 -d '{"redirect_uri":"https://connect.service.ab0t.com/connections/google_calendar/oauth/callback"}'
# -> { "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth?.." }

# 2. a HUMAN opens authorization_url and approves (this is the only human step)
# 3. provider redirects to the callback; the service exchanges code -> tokens -> stores the connection
# 4. agent polls until the connection appears, then uses it forever
curl $URL/connections/ -H "X-API-Key: $KEY" | jq '.[]|select(.service_id=="google_calendar")'
```
`connect add google_calendar` automates steps 1–4 (opens the browser, polls). Note: the api-key endpoint **strips** `access_token`/`refresh_token`, so you cannot smuggle OAuth tokens through it.

## Patterns for a fully agentic system (pick per service + trust model)

| Pattern | Human step | Best for | Notes |
|---------|-----------|----------|-------|
| **One-time consent + reuse** (recommended) | once, ever | any OAuth service, per end-user | Human/operator authorizes once via the URL; the connection auto-refreshes; agents act headless thereafter. The service is built for this — the connection IS the agent's standing authorization. |
| **Operator pre-provision** | once, by you | shared/org accounts | An operator connects a team Google/Slack account; all agents in the org use that `connection_id`. |
| **Service account / domain-wide delegation** (Google Workspace) | zero (admin sets it up once) | Workspace orgs, no per-user popup | Requires a `service_account` connection type + JSON key + domain admin granting scopes. Fully headless, no per-user consent. **Not implemented today** — would add a `service_account` auth type to the service. |
| **OAuth Device Authorization Grant (RFC 8628)** | once, enter a code | headless/CLI/no-browser agents | Agent shows `user_code` + URL; human enters it on any device. No browser on the agent box. **Not implemented today** — would add device-flow to the OAuth handler. |
| **Direct refresh-token injection** | obtained out-of-band | migrations, BYO-token | A privileged/admin endpoint that stores a pre-obtained `refresh_token` as a connection. **Intentionally blocked on the public api-key endpoint** (reserved keys stripped); would need a dedicated admin path with strong authz. |
| **Client-credentials grant** | zero | service-to-service, app-owned data | Works only for APIs that support app-level (non-user) auth; useless for Gmail/Calendar user data. |

### Recommended architecture
1. **API-key services:** agents self-connect with the secret. Done.
2. **OAuth user-data services:** a one-time human consent per account (URL hand-off works fine in an agent loop — the agent emits the `authorization_url`, a human/HITL approves, the agent resumes on the new `connection_id`). This is the normal, secure path and keeps humans in control of granting access.
3. **For zero-human Workspace/G-Suite automation:** add a **service-account connection type** (domain-wide delegation) — the only genuinely consent-free path for Google user data, set up once by an admin.
4. **Always:** agents discover and reuse connections via `GET /connections/`; never re-consent if a valid connection exists.

### Minimal agent connect loop (pseudocode)
```python
conns = GET("/connections/")
conn = next((c for c in conns if c["service_id"]=="google_calendar" and c["status"]=="active"), None)
if not conn:
 if service_is_api_key("google_calendar"):
 conn = POST(f"/connections/google_calendar/api-key", {"api_key": secret})
 else:
 url = POST("/connections/google_calendar/oauth/authorize", {"redirect_uri": CALLBACK})["authorization_url"]
 ask_human(url) # the one human touch (HITL); or skip if service-account
 conn = poll_until_connection("google_calendar")
# act, headless, forever:
POST(f"/uts/v1/tools/google-calendar/create_event/execute", {"connection_id": conn["connection_id"], "input": {..}})
```
