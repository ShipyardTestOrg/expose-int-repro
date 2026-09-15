# expose-int-repro

A deliberately minimal service used to reproduce two Shipyard configure-page bugs
on QA. Both bugs are triggered by **valid** compose files that Shipyard
mishandles — `docker compose config` accepts every branch here without complaint.

The app itself is a ~15-line Python HTTP server on port 8080 so builds are fast
and nothing else can be blamed for a failure.

## Branches

| Branch | Compose quirk | What breaks | Fixed by |
|---|---|---|---|
| `main` | `expose:` entries unquoted (`- 9090`) | Configure page renders blank | shipyard#3949 |
| `tmpfs-volume` | long-syntax volumes with no `source` | Shipyard rejects the file as invalid | not yet fixed |
| `control` | quoted `expose`, volumes with a `source` | nothing — baseline | n/a |

## Why each one breaks

### `main` — unquoted `expose`

```yaml
expose:
  - 9090
```

YAML parses these as integers. `ComposeService.ports` coerces with `str()`, but
`expose` returned the raw parsed value, and the configure endpoint folds `expose`
into the `ports` it sends the frontend. The frontend then calls `port.match(...)`
on each entry and throws `TypeError: port.match is not a function` during render.
That throw happens above the page's error boundary, so React unmounts the whole
root and the page goes blank rather than showing a fallback.

Verified against the Shipyard parser on master:

```
service_ports = [9090, '8080', 9091]   # ints, and set() scrambled the order
```

### `tmpfs-volume` — long-syntax volume with no `source`

```yaml
volumes:
  - type: tmpfs
    target: /tmp
```

This is valid Docker Compose — `docker compose config` accepts it — but Shipyard
**rejects the whole file**:

```
ShipyardBuildError / BuildFailure.INVALID_COMPOSE_FILE
Invalid volumes entry in service 'web': '{'type': 'tmpfs', 'target': '/tmp'}' must be a string.
```

`validate_compose_format` requires every entry in `volumes:` to be a string
(`shipyard/models/compose.py`, "must be a string"), so the long syntax never
parses at all.

**This branch does not reproduce the `KeyError('source')` that shipyard#3951
fixes.** That crash lives further down in `ComposeService._volumes`, which this
input never reaches. The KeyError is only reachable for Compose rows stored
*before* the validation check landed (2025-01-28, shipyard#3378) — i.e. legacy
data, not anything you can create today.

What this branch is genuinely useful for is the larger bug underneath: Shipyard
refuses valid Docker Compose. On QA, `/api/git-provider/<uuid>/<org>/<ns>/expose-int-repro/compose?branch=tmpfs-volume`
returns **404** with the message above.

### `control` — the same service written the boring way

Verified on master: `service_ports = ['9091', '8080', '9090']`, all strings,
`all_volumes = [('scratch', '/var/lib/scratch')]`. Use it to confirm the page
works for an ordinary project, so a failure elsewhere isn't mistaken for an
environment problem.

Note the order here is still scrambled — `service_ports` used `list(set(...))`,
which reshuffles the API response between requests. #3949 makes it order-stable.

## QA test plan

Order matters: bug 1 blanks the page, so you can't reach the branch switcher to
test bug 2 until #3949 is deployed.

**Setup** — create an application in the QA org from this repo, branch `main`.

**1. Reproduce the crash (before #3949)**

Open `/application/<uuid>/configure/repos`. Expect a blank page and, in the
console, `Uncaught TypeError: e.match is not a function`. Expect **nothing** in
Sentry — that page has no Sentry client until #3950.

**2. Verify the fix (after #3949)**

Same URL. The page renders, and the service shows ports 8080, 9090 and 9091.

**3. Verify Sentry coverage (after #3950)**

Reload the page and confirm a Sentry envelope request goes out on load, or that
`window.__SENTRY__` is populated. Previously only 4 of ~48 pages had a client.

**4. Long-syntax volumes are rejected (not yet fixed)**

Switch the branch to `tmpfs-volume`, or request the compose endpoint directly.
Expect a **404** and `Invalid volumes entry ... must be a string`. This is a
standing bug, not something the three PRs address — see the branch notes above.

**Baseline at any point** — switch to `control`; it should always work.
