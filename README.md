# polygate-loop

A self-dispatching GitHub Actions relay that keeps a [Polygate](https://github.com/B3hnamR/Polygate)
panel online on a fresh GitHub runner, under one stable domain:

```
runner a  --dispatch-->  runner b  --dispatch-->  runner a  --...
```

- **Panel:** `https://polygate.codol.ir/`
- **API base:** `https://polygate.codol.ir/v1`

Every run is a brand-new VM; the panel URL, the dashboard password and the
client API key stay the same.

## How it works

| Piece | Role |
| --- | --- |
| `.github/workflows/polygate-loop.yml` | installs Polygate on the runner, opens a Cloudflare Tunnel, serves for ~5h15m, queues the other half, stops |
| `.github/workflows/polygate-watchdog.yml` | every 30 minutes: if no run is queued/running, dispatch `runner=a` |
| `scripts/runner-polygate/polygate.mjs` | the actual start/stop script (tunnel + install + seed + checks) |

```
cloudflared  ->  127.0.0.1:4317   the public entry for <subdomain>.codol.ir
polygate     ->  systemd (user "polygate"), state in /var/lib/polygate
```

The relay is one concurrency group (`polygate-loop`), so exactly one gateway is
alive at a time. The next run is queued a few seconds **before** the current one
ends and starts on a fresh runner right after it. A hard kill (cancel, 6h cap,
failed tunnel) is covered by the watchdog.

## Fixed values

Only two things are pinned, both repository secrets:

| Secret | Meaning |
| --- | --- |
| `POLYGATE_ADMIN_PASSWORD` | dashboard password (12+ characters) |
| `POLYGATE_CLIENT_KEY` | the client API key (`pg_...`) |

Polygate generates key values itself and its API does not accept a chosen value,
so the fixed key is carried by a tiny encrypted **seed**:

| Secret | Meaning |
| --- | --- |
| `POLYGATE_SEED_B64` | base64 of a Polygate settings export holding exactly that one key |
| `POLYGATE_SEED_PASSPHRASE` | passphrase of that encrypted seed |

The seed is restored on every start (`POST /admin/backup/restore`) and then
`GET /v1/models` proves the fixed key is accepted. There is **no state backup,
no snapshot and no export**: each session starts from the seed and dies with the
runner. Dashboard changes (providers, routes, chains, extra keys) last for the
current session only.

## Running it

Actions -> **polygate-loop** -> *Run workflow*:

| Input | Default | Notes |
| --- | --- | --- |
| `runner` | `a` | which half starts; it only decides who is dispatched next |
| `subdomain` | `polygate` | first label only, `.codol.ir` is appended |
| `session_seconds` | `18900` | 5h15m; GitHub caps a job at 21600 (6h) |

Use a subdomain that no other session is using: the DNS record is repointed to
this run's tunnel.

## Required secrets

`POLYGATE_ADMIN_PASSWORD`, `POLYGATE_CLIENT_KEY`, `POLYGATE_SEED_B64`,
`POLYGATE_SEED_PASSPHRASE`, `CF_API_TOKEN` (Zone:DNS:Edit + Cloudflare Tunnel:Edit)
, `CF_ZONE_ID`, and `GH_PAT` (a token with `repo` + `workflow`) so a run can
dispatch the next one and the watchdog can restart the relay.

## Notes

- The panel has no inbound port: cloudflared dials out, and the tunnel + DNS are
  created/reused through the Cloudflare API on every start.
- Polygate brings its own exits (Direct, Cloudflare WARP, proxy pool, relays);
  pick one per provider on the **Routing** page.
- Running a public service on free GitHub runners around the clock is outside
  GitHub's acceptable use; treat this as a testing setup.
