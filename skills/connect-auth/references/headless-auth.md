# Headless / agent / CI auth

> The CLI is `connect` (env `CONNECT_*` →
> `CONNECT_*`). Substitute the new names once the rename lands.

The goal of headless auth: **no browser, no interactive prompt, no on-disk state
mutation.** An API key (an `ab0t_sk_live_…` mesh key) is the only mode that
satisfies all three. JWT user login can't — it needs a one-time browser consent.

## The core pattern

Hold the key in the environment; the CLI reads it on every call and presents it
as `Authorization: Bearer <key>` (the mesh also accepts `X-API-Key`):

```bash
export CONNECT_API_KEY=ab0t_sk_live_..
connect whoami # verifies the key reaches the mesh and resolves an identity
connect services list # any command now runs authenticated
```

`CONNECT_API_KEY` **wins over** any credential stored for the active profile, so
a CI runner with the env var set acts as the key's identity regardless of what's
on disk. To keep a run from touching disk at all, inline the var on the command:

```bash
CONNECT_API_KEY=ab0t_sk_live_.. connect tools catalog
```

## Storing the key instead (optional)

If you'd rather not carry the env var, log the key into a profile once:

```bash
connect login --api-key ab0t_sk_live_.. # stores it in the active profile's credential file
connect whoami
```

After this, plain `connect …` commands use the stored key. Use `--profile <name>`
to write it into a specific profile (e.g. a dedicated `ci` profile) without
disturbing the active one:

```bash
connect --profile ci login --api-key ab0t_sk_live_..
connect --profile ci services list
```

## Picking the host in CI without profiles

CI often has no saved profiles. Resolution order is **flag → env → profile →
default**, so set the host the same way you set the key — no `profile create`
needed:

```bash
export CONNECT_API_URL=https://connect.service.ab0t.com # or your target mesh
export CONNECT_API_KEY=ab0t_sk_live_..
connect services list
```

Or per-command: `connect --api-url https://… --profile ci services list`.

## Machine-readable output for agents

Pair headless auth with structured output so an agent can parse results:

```bash
CONNECT_API_KEY=ab0t_sk_live_.. connect whoami -o json
```

`-o json` (or `CONNECT_OUTPUT=json`) emits JSON instead of tables. Suppress the
human NEXT-step prose on stderr with `--no-hints` / `CONNECT_NO_HINTS`.

## A minimal agent auth check

Before doing real work, an agent should confirm it's authenticated and pointed at
the right host, then branch on the result:

```bash
if CONNECT_API_KEY=$KEY connect whoami -o json >/tmp/who.json 2>/dev/null; then
 : # authenticated — proceed
else
 : # key missing/invalid/expired — rotate the key in the mesh and retry
fi
```

A `401` here means the key is missing, malformed, expired, or its audience
doesn't match the host you resolved. Rotate the key in the mesh, re-export, and
re-run. Confirm the host you actually hit with `connect --verbose whoami` (prints
the resolved profile / API URL / auth URL on stderr).

## Rotating keys

API keys don't expire on a fixed clock the way JWTs do — you rotate them
deliberately in the mesh. After minting a replacement:

```bash
export CONNECT_API_KEY=ab0t_sk_live_<new> # ambient: just re-export
# or, if stored:
connect logout --purge && connect login --api-key ab0t_sk_live_<new>
```

`--purge` permanently deletes the stored credential (vs the default archive),
which is the right choice when retiring a rotated key so a later `--restore`
can't resurrect it.
