---
name: connect-auth
description: >-
 Set up and manage the connect mesh CLI's credentials and which environment it
 points at. Use when you need to log in / authenticate the CLI, create or use an
 API key, work with a JWT user token, manage profiles, switch environment (dev
 vs prod host) or org, do headless / agent / CI auth, or override the host per
 command. Triggers: "log in", "authenticate the CLI", "API key", "JWT bearer",
 "Authorization Bearer", "X-API-Key", "profiles", "profile show/add/use",
 "switch environment", "dev vs prod host", "point at a different host", "org
 slug", "headless auth", "agent auth", "CI auth", "CONNECT_API_KEY", "whoami",
 "logout", "not logged in", "401 unauthorized", "~/.connect config",
 "config dir", "--api-url override", "--auth-url override", "--profile flag".
---

> The CLI is `connect` (config dir `~/.config/connect`, env `CONNECT_*`).
> Public host: `connect.service.ab0t.com` (dev: `connect.dev.ab0t.com`).

The CLI talks to the connect mesh over HTTPS. Every authenticated request carries
`Authorization: Bearer <token>` (the mesh also accepts `X-API-Key: <key>`). The
token is either a **JWT user token** (obtained by logging in, auto-refreshed) or
an **API key** (an `ab0t_sk_live_…` mesh key, fully headless). Public / discovery
endpoints need no auth at all.

A **profile** decides which host you point at and which org you act as. Profiles
keep environments isolated — each has its own API URL, auth URL, org slug, and
its own stored credentials.

## Two auth modes — pick by who's running the CLI

| Mode | Who | How | Headless? |
| --- | --- | --- | --- |
| **JWT user token** | a human | `connect login` (browser consent) | no — needs a browser once |
| **API key** | agents, CI, services | an `ab0t_sk_live_…` key | yes — no browser, ever |

Both end up as `Authorization: Bearer <token>`. The mesh tells them apart by the
token shape; you never set the header yourself.

### JWT user login (humans)

```bash
connect login # browser OAuth consent, then stores the token
connect auth whoami # confirm identity / org / token validity
connect auth token # print the raw access token (for piping)
connect logout # archives the credential (restorable)
connect login --restore # bring an archived credential back, no key re-entry
connect logout --purge # permanently delete (e.g. on key rotation)
```

Browser login needs an org slug AND an OAuth client id; on known environments the
CLI resolves both defaults, so `connect login` works zero-flag there. Where it
can't, it prints a chooser. The JWT carries a refresh token and is **refreshed
automatically** on each call — you don't manually re-auth until the refresh
itself expires (then `connect login` again).

### API key (agents / CI / headless)

An API key needs no browser and no org slug. Two equivalent ways to use it:

```bash
# 1) Ambient — the CLI reads the key from the environment every call.
export CONNECT_API_KEY=ab0t_sk_live_..
connect whoami # the key reaches the mesh; you're authed

# 2) Stored — log the key into the active profile once, then run plain commands.
connect login --api-key ab0t_sk_live_..
connect whoami
```

The ambient `CONNECT_API_KEY` env var **takes precedence** over any stored
credential for the active profile — ideal for one-shot agent / CI invocations
that shouldn't mutate on-disk state:

```bash
CONNECT_API_KEY=ab0t_sk_live_.. connect services list
```

API keys do not expire on a timer the way JWTs do; rotate them in the mesh and
re-export. See [references/headless-auth.md](references/headless-auth.md) for the
full agent / CI pattern.

## Profiles — environment + org switching

A profile bundles `api_url`, `auth_url`, and `org_slug`, plus its own credential
file. Switching profiles is non-destructive — it never logs you out of the others.

```bash
connect profile show # the active profile's host, auth URL, org, auth state
connect profile list # all profiles (* marks active) + each one's auth state
connect profile create staging --org-slug acme-staging # add a profile (this skill: "profile add")
connect profile use staging # switch active profile
connect profile delete staging # remove a profile + its stored credential
```

> The standard subcommand to create a profile is `profile create`. References to
> "profile add" mean the same operation.

Create a profile pointed at a non-default host, then authenticate it:

```bash
connect profile create dev --api-url https://dev-host.example --auth-url https://dev-auth.example
connect --profile dev login # auth THIS profile without switching the active one
connect profile use dev # ..or make it the active profile
```

Each profile authenticates independently: a `dev` profile can hold a JWT while
`production` holds an API key. `connect profile list` shows each profile's auth
state side by side.

## Where the host comes from — resolution order

For both the API URL and the auth URL, the CLI resolves **first match wins**:

```
1. command flag --api-url / --auth-url
2. environment CONNECT_API_URL / CONNECT_AUTH_URL
3. active profile api_url / auth_url
4. built-in default
```

Default API URL: `https://connect.service.ab0t.com` (dev: `https://connect.dev.ab0t.com`).
Use the overrides to hit one mesh without disturbing your saved profiles:

```bash
connect --api-url https://other-host.example services list
CONNECT_API_URL=https://other-host.example connect services list
```

`--profile <name>` is a global flag too — run any command against a named
profile's credentials without `profile use`.

## Config & credentials on disk

```
~/.config/connect/config.yaml # current_profile + per-profile api_url/auth_url/org_slug
~/.config/connect/credentials-<profile>.json # the stored token/key for that profile (0600)
```

(Set `CONNECT_CONFIG_DIR` to relocate the whole directory.) Config files are
written `0600`; credential files hold the live token — treat them as secrets.


## Quick decisions

- **"It says I'm not logged in."** Run `connect login` (human) or set
 `CONNECT_API_KEY` (agent). Check the active profile first with
 `connect profile show` — you may be authed on a *different* profile.
- **401 unauthorized.** Token expired / wrong audience for this host. Re-`login`,
 or rotate + re-export the API key; confirm with `connect auth whoami`.
- **Wrong environment.** `connect profile use <name>`, or one-shot with
 `--api-url` / `CONNECT_API_URL`.
- **Agent / CI run.** `export CONNECT_API_KEY=…` (or inline it on one command).
 No browser, no profile mutation. See
 [references/headless-auth.md](references/headless-auth.md).
