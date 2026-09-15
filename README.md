# expose-int-repro

Reproduces two Shipyard configure-page bugs with valid-but-unusual compose files.

## `main` — unquoted `expose` ports

`docker-compose.yml` writes `expose` entries unquoted, so YAML parses them as
integers:

```yaml
expose:
  - 9090
```

The configure endpoint folds `expose` into the `ports` it returns, and the
frontend calls `port.match(...)` on each one. A number there throws
`TypeError: port.match is not a function` during render, which unmounts the page.

**Expected (broken):** `/application/<uuid>/configure/repos` renders blank.

## `tmpfs-volume` — long-syntax volume with no `source`

`source` is optional in the compose long syntax. `ComposeService._volumes`
indexed it unconditionally and raised `KeyError('source')`, 500ing the
git-provider compose endpoint and breaking branch selection.

**Expected (broken):** changing the branch on the configure page fails.
