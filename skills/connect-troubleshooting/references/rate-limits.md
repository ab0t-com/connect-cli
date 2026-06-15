# Connect mesh — rate limits & backoff

> CLI is `connect`. Default host
> `connect.service.ab0t.com`.

How rate limiting works across the Connect mesh, the headers that report it, and
how to back off correctly. The [`SKILL.md`](./SKILL.md) has the diagnostic flow;
[`error-reference.md`](error-reference.md) has the full code table.

## How rate limiting works

The mesh **mediates** every call it makes on your behalf — proxy passthroughs and
UTS tool executions both run through it. Two limits can apply:

1. **The mesh's own quota** for the call (it rate-limits and audits each request).
2. **The upstream provider's quota** for that account — the provider's limit on
 the connected credential, surfaced through the passthrough.

Because the mesh sits in the middle, a `429` you receive may originate at either
layer. The handling is the same: read the headers, back off, retry. When a `429`
carries an upstream **provider** body on a `/proxy/…` call, the provider throttled
you — honor the provider's own `Retry-After`.

## The headers

The mesh adds these to responses (alongside `X-Integration-Service`,
`X-Integration-Response-Time`, `X-Request-ID`):

- **`X-Integration-Rate-Limit`** — appears when the mesh is tracking rate-limit
 state for the call. Use it to see how close you are to a limit *before* you trip
 one.
- **`X-Integration-Cost`** — may accompany it when the mesh tracks per-call cost.
- **`Retry-After`** — on a `429`, the number of seconds to wait before retrying.
 Honor it as the floor of your backoff.

UTS tool errors also carry the limit structurally:
`{status:429, code:"RATE_LIMITED", retryable:true, retry_after:60}`. Branch on
`retryable` and sleep `retry_after` rather than guessing.

## Handling a 429

1. **Do not retry immediately.** A tight retry loop makes it worse.
2. **Read `Retry-After`** (or the tool error's `retry_after`). Wait at least that
 long.
3. **If no header is present, fall back to exponential backoff** (below).
4. **Retry the same call.** Capture `X-Request-ID` if it keeps failing — quote it
 to support.

## Exponential backoff guidance

For `429` (and for retryable network errors like `502`), back off exponentially
with jitter and a cap:

```
attempt 1: wait base (e.g. 1s)
attempt 2: wait base * 2
attempt 3: wait base * 4
..
wait = min(cap, base * 2**(attempt-1)) + random_jitter
```

- **Floor at `Retry-After`** when present — never retry sooner than the server
 asked.
- **Add jitter** (a small random fraction) so many clients don't retry in lockstep.
- **Cap the delay** (e.g. 30–60s) and **cap attempts** (e.g. 5) — then surface the
 failure rather than looping forever.
- **Only retry retryable failures.** `429` and `502` are retryable;
 `400`/`403`/`404`/`422` are not — fix the request instead (see
 [error-reference.md](error-reference.md)).

```python
import time, random

def with_backoff(call, max_attempts=5, base=1.0, cap=30.0):
 for attempt in range(1, max_attempts + 1):
 resp = call()
 if resp.status_code != 429:
 return resp
 retry_after = float(resp.headers.get("Retry-After", 0))
 backoff = min(cap, base * 2 ** (attempt - 1)) + random.uniform(0, base)
 time.sleep(max(retry_after, backoff)) # floor at the server's hint
 return resp # give up; surface the 429 + its X-Request-ID
```

## Reducing pressure

- **Batch** UTS tool calls (`POST /uts/v1/tools/batch`) instead of firing many
 single executes back-to-back.
- **Respect provider pagination** — the proxy doesn't auto-paginate; pull pages at
 the provider's pace, honoring its `Link`/cursor headers.
- **Spread multi-account work** across the right `connection_id`s rather than
 hammering one connection's provider quota.
- **Watch `X-Integration-Rate-Limit`** to throttle *before* you hit `429`.
