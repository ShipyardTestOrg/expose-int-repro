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
| `tmpfs-volume` | long-syntax volumes with no `source` | Branch switching 500s | shipyard#3951 |
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

`source` is optional in the compose long syntax — tmpfs mounts and anonymous
volumes have a target only. `ComposeService._volumes` indexed `v['source']`
unconditionally. The git-provider compose endpoint serializes `all_volumes`, and
its `except ShipyardException` doesn't catch a `KeyError`, so the request 500s
and branch selection breaks on the add and configure pages.

Verified against the Shipyard parser on master:

```
KeyError('source')
```

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

**4. Reproduce the branch-switch 500 (before #3951)**

On the configure page, switch the branch from `main` to `tmpfs-volume`. Expect
the compose request to fail — a 500 from
`/api/git-provider/.../compose`, with `KeyError: 'source'` in the web logs.

**5. Verify the fix (after #3951)**

Switch to `tmpfs-volume` again. The services load, with the source-less volumes
omitted from the volume list.

**Baseline at any point** — switch to `control`; it should always work.
