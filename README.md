# polygate-loop

A self-dispatching GitHub Actions relay that keeps a [Polygate](https://github.com/B3hnamR/Polygate)
panel online on fresh GitHub runners, under one stable domain:

```
runner a --(overlap)--> runner b --(overlap)--> runner a --...
```

- **Panel:** `https://polygate.codol.ir/`
- **API base:** `https://polygate.codol.ir/v1`

Every run is a brand-new VM; the panel URL, the dashboard password and the
client API key stay the same.

## Timing

| | |
| --- | --- |
| Planned life of a run | `session_seconds = 3900` (65 minutes) |
| Next run dispatched | 5 minutes before that end (at minute 60) |
| Old run stops | only after the new panel is confirmed up |
| Fresh run cadence | **once an hour** |
| Overlap | ~1 minute, two runners on the same tunnel |

The new runner boots, installs Polygate, restores the seed and binds the tunnel
while the old one is still serving. The old run waits for the new run's
"Start Polygate and the tunnel" step to succeed (and then 20 more seconds),
and only then stops its own gateway and cloudflared.

## How it works

| Piece | Role |
| --- | --- |
| `.github/workflows/polygate-loop.yml` | installs Polygate, opens the tunnel, serves 65 min, brings the other half up, hands over, stops |
| `.github/workflows/polygate-watchdog.yml` | every 30 minutes: if no run is queued/running at all, dispatch `runner=a` |
| `scripts/runner-polygate/polygate.mjs` | start/stop script (tunnel + install + seed + checks) |

```
cloudflared  ->  127.0.0.1:4317   the public entry for <subdomain>.codol.ir
polygate     ->  systemd (user "polygate"), state in /var/lib/polygate
```

There is deliberately **no shared concurrency group**: the relay overlaps for a
few minutes and a manual run must start at once instead of queueing. Duplicate
chains are prevented at the handoff instead: a run only dispatches when no newer
run for the same subdomain is already active, and the older of two runs stops
itself after the newer one is up.

## Manual override

Two cases this is built for:

1. **Quota ran out right now.** Start a run from Actions -> **polygate-loop** ->
   *Run workflow*. It starts immediately (no queue), and it is the newest run,
   so the previous run stops by itself once the new panel is confirmed.
   Cancelling the old run by hand works too and is safe.
2. **Starting a second panel** on another subdomain. Each run only looks at runs
   with its own subdomain, so two relays do not fight each other.

| Input | Default | Notes |
| --- | --- | --- |
| `runner` | `a` | which half starts; it only decides who is dispatched next |
| `subdomain` | `polygate` | first label only, `.codol.ir` is appended |
| `session_seconds` | `3900` | next run is dispatched 5 min before this deadline |

Use a subdomain that no other run is using: the DNS record is repointed to this
run's tunnel.

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

## Required secrets

`POLYGATE_ADMIN_PASSWORD`, `POLYGATE_CLIENT_KEY`, `POLYGATE_SEED_B64`,
`POLYGATE_SEED_PASSPHRASE`, `CF_API_TOKEN` (Zone:DNS:Edit + Cloudflare Tunnel:Edit),
`CF_ZONE_ID`, and `GH_PAT` (a token with `repo` + `workflow`) so a run can
dispatch the next one and the watchdog can restart the relay.

## Notes

- The panel has no inbound port: cloudflared dials out, and the tunnel + DNS are
  created/reused through the Cloudflare API on every start. An already-correct
  DNS record is left untouched so a handoff never flaps DNS.
- Polygate brings its own exits (Direct, Cloudflare WARP, proxy pool, relays);
  pick one per provider on the **Routing** page.
- Running a public service on free GitHub runners around the clock is outside
  GitHub's acceptable use; treat this as a testing setup.
