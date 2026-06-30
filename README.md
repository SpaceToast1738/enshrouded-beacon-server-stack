# enshrouded-beacon-server-stack

Docker Compose deployment for an [Enshrouded][game] dedicated
server **plus** the [enshrouded-beacon][source] companion daemon
that forwards world stats to your dashboard. One command brings
both up; Watchtower keeps both auto-updated.

[game]: https://enshrouded.com
[source]: https://github.com/spacetoast1738/enshrouded-beacon

## What this stack runs

```
┌─────────────────────── your Ubuntu host ───────────────────────┐
│                                                                │
│  ┌────────────────────┐    ┌────────────────────────┐          │
│  │ enshrouded-server  │    │ enshrouded-beacon      │          │
│  │ Wine + SteamCMD    │    │ (this monorepo's       │          │
│  │ community image    │    │  server-companion)     │          │
│  │                    │    │                        │          │
│  │ savegame/ + logs/  ├───►│  /save:ro   /logs:ro   │          │
│  │ + enshrouded_      │    │  /state:rw             │          │
│  │   server.json      │    │                        │          │
│  └────────────────────┘    └────────────────────────┘          │
│         UDP 15636/7/8                │                         │
│             ▼                        │ HTTPS POST /ingest      │
│         (Players)                    │                         │
│                                      │                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ watchtower  (auto-pulls :latest tags every 5 min)        │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────┬───────────────────────┘
                                         │
                                         ▼
                          (your separate dashboard host)
```

Three containers:

1. **`enshrouded-server`** — community Wine-based image
   ([`sknnr/enshrouded-dedicated-server`][sknnr]) running
   the real Windows dedicated-server binary under Wine on
   Linux. SteamCMD-managed; auto-downloads the game server
   on first up.
2. **`beacon`** — our published
   `ghcr.io/spacetoast1738/enshrouded-beacon-server:latest`.
   Tails the dedicated server's save files + logs (read-only)
   and POSTs world stats every 30 seconds to your dashboard
   via HTTPS.
3. **`watchtower`** — auto-updates both images by polling
   `:latest` tags every 5 minutes. Disable if you prefer
   manual upgrades.

The dashboard itself **is not part of this stack** — it lives
on a separate host (typically your home server / Unraid box /
VPS). This stack only forwards data outbound; the dashboard
handles storage, presentation, and admin.

[sknnr]: https://github.com/jsknnr/enshrouded-server

## Prerequisites

- **Ubuntu 22.04+** (or any modern Linux with kernel 5.4+).
  Other distros work too; commands below assume Ubuntu.
- **Docker Engine 24+** with the Compose plugin v2+
  (`docker compose`, not the old `docker-compose`). Install
  recipe below for fresh VPSes.
- **A reachable dashboard host** running
  [enshrouded-beacon][source] 0.43.0+. HTTPS strongly
  recommended (your dashboard's Cloudflare tunnel /
  reverse-proxy handles TLS).
- **A plaintext ingest token** issued from that dashboard's
  `/admin → Dedicated servers → Register a new dedicated
  server`. Shown once at issue time. See
  [`docs/token-setup.md`](docs/token-setup.md).
- **UDP ports 15636-15638 open** on the host's firewall.
  See [`docs/port-forwarding.md`](docs/port-forwarding.md).
- **~5 GB free disk** for the initial SteamCMD download +
  Wine prefix. World saves grow gradually beyond that.

### Installing Docker on a fresh Ubuntu host

Skip this section if `docker compose version` already prints
something. Otherwise, from the host's terminal:

```bash
# Official install script — handles all current Ubuntu releases
# including 25.04 'plucky' which isn't yet in Docker's static
# apt repo at the time of writing.
curl -fsSL https://get.docker.com | sudo sh

# Add yourself to the docker group so you don't need sudo every
# call. `newgrp docker` picks up the new group in THIS shell
# without forcing a logout.
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker compose version
```

You should see `Docker Compose version v2.x.x`. If
`docker compose` returns "command not found" instead, the
script picked up the legacy `docker.io` apt package instead of
Docker CE — uninstall it (`sudo apt remove docker.io`) and
re-run the get.docker.com script.

> **Snap warning.** Ubuntu's `apt search docker` suggests
> `snap install docker` first. **Don't.** The snap version
> sandboxes container bind-mounts in ways that break this
> stack's `./data/` mounts. Use the official Docker CE repo
> (get.docker.com handles this).

## Quickstart

```bash
git clone https://github.com/spacetoast1738/enshrouded-beacon-server-stack
cd enshrouded-beacon-server-stack

cp .env.example .env
nano .env             # fill in DASHBOARD_URL, ESB_TOKEN,
                      # SERVER_NAME, optional SERVER_PASSWORD
                      # (substitute your editor of choice;
                      # $EDITOR is unset on minimal installs)

# Pre-create the bind-mount dirs owned by the dedicated-server
# image's runtime user (uid 10000 = `steam` inside
# sknnr/enshrouded-dedicated-server). Skipping this step means
# Docker auto-creates the dirs as root:root and the server
# fail-loops with "cannot create file ... Permission denied"
# on every restart — re-downloading ~8 GB each cycle.
mkdir -p data/enshrouded data/beacon-state
sudo chown -R 10000:10000 data/enshrouded data/beacon-state

docker compose up -d
docker compose logs -f beacon          # watch the first /ingest post
```

That's it. On first up the `enshrouded-server` container
downloads the game server via SteamCMD (~3 GB; expect 5-10
minutes). Once it's ready, the `beacon` container starts posting
to your dashboard within 30 seconds, and the row appears under
`/admin → Dedicated servers`.

## Verifying it works

After `docker compose up -d`:

1. **Server boots** — `docker compose logs -f enshrouded-server`
   should reach a line containing
   `Loaded game server` / `Game server fully initialised` (the
   exact phrasing depends on the community image's logging).
   ~5-10 min on first up while SteamCMD downloads + Wine
   bootstraps; ~30 seconds on subsequent restarts.
2. **Beacon registers** — within 30 seconds of `up -d`, the
   beacon's first POST hits your dashboard. Open `/admin →
   Dedicated servers` and confirm a new row with the name you
   set in `SERVER_NAME`, recent `last_seen`, and a populated
   `world_id`.
3. **Wire log surfaces it** — open `/admin → Dedicated-server
   wire log`, pick this server, and you'll see the raw payload
   under "Show".
4. **First save tick** — after ~5 minutes of in-game activity
   (the dedicated server saves every 5 min by default), the
   `last_save_tick_ts` column starts advancing and per-tag
   compression times appear inside `current_state.client_state.
   save_tick_compression_us`.
5. **Players can join** — connect to the server's external IP
   (from inside or outside the host's network) with the in-game
   client. If you can't, see [`docs/port-forwarding.md`](docs/port-forwarding.md).

## Updating

**Watchtower auto-updates** are on by default. Every 5 minutes
it polls `:latest` for both images and re-pulls anything new.
You can watch it via:

```bash
docker compose logs -f watchtower
```

**Manual updates**:

```bash
docker compose pull
docker compose up -d
docker compose images           # confirm new digests
```

**Tag-pinning** for production stability: edit
`docker-compose.yml`, replace `:latest` with a specific tag
(e.g. `:0.43.1` for the beacon), comment out the `watchtower`
service, then `docker compose up -d`. Updating becomes a manual
edit-and-up cycle.

See [`docs/upgrading.md`](docs/upgrading.md) for more detail.

## Backups

The single critical directory is `./data/enshrouded/savegame/`.
Everything else (Wine prefix, SteamCMD cache, downloaded server
binary) regenerates on `docker compose up -d`.

A daily tarball cron is the simplest backup:

```bash
# /etc/cron.daily/enshrouded-backup (chmod +x)
#!/bin/bash
cd /opt/enshrouded-beacon-server-stack
ts=$(date +%Y%m%d-%H%M)
tar -czf "/var/backups/enshrouded/savegame-${ts}.tar.gz" \
    data/enshrouded/savegame
find /var/backups/enshrouded -mtime +14 -delete   # keep 14 days
```

## Troubleshooting

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for
symptom-by-symptom diagnosis. Most common:

- **Beacon container restarts in a loop** → token mismatch.
  Check `ESB_TOKEN` matches what `/admin` shows. Wire log
  endpoint will return 401s.
- **Server fails to start, can't find Steam** → SteamCMD
  download failure or Wine prefix corruption. See the
  community image's [troubleshooting docs][sknnr-tshoot].
- **Beacon says "no log found"** → check
  `data/enshrouded/logs/enshrouded_server.log` exists. If
  the community image uses a non-standard path, override
  with `ESB_LOG_PATH` in `.env`.
- **Players can't connect** → UDP ports not open. See
  [`docs/port-forwarding.md`](docs/port-forwarding.md).

[sknnr-tshoot]: https://github.com/jsknnr/enshrouded-server#troubleshooting

## Diagnostic bundles (0.44.0+, opt-in)

The dashboard maintainer can optionally request raw save +
config + log bundles from your beacon for offline reverse-
engineering + catalog mining work. **Off by default** —
set `ESB_ALLOW_DIAGNOSTIC_BUNDLES=1` in `.env` to enable.

When disabled (default), the routine `/ingest` wire payload
continues exactly as before; the new file-pull workflow is
gated behind the env var. When enabled, the maintainer's
"Request bundle" click triggers a one-off upload of a
gzipped tarball (≤ 10 MB) containing the active save slot,
redacted config, log tail, and optionally selected game
catalog `.dat` files.

See [`docs/diagnostic-bundles.md`](docs/diagnostic-bundles.md)
for the full content list, privacy details, and revocation
path.

## Edit server settings from the website (0.90.0+, opt-in)

From dashboard + beacon **0.90.0**, admins can optionally edit
this server's `gameSettings` (difficulty multipliers, enemy/XP
factors, day-night cycle, etc.) **and restart the server** to
apply them — from the web UI, no SSH. **Off by default.**

⚠ It's opt-in at several layers, two of which are easy to miss
and cause **silent failure** if skipped:

- **`EXTERNAL_CONFIG=1`** on the dedicated-server service — the
  community image regenerates `enshrouded_server.json` on every
  boot by default, which would wipe dashboard edits on restart.
- A **`chown` of the config file** to the beacon's uid (10100)
  plus flipping its bind-mount to `:rw`.
- **`ESB_ALLOW_SETTINGS_WRITE=1`** (and, for restart-from-website,
  the opt-in `docker-proxy` service + `ESB_ALLOW_SERVER_RESTART=1`).

The save dir stays read-only throughout — only the single config
*file* becomes writable, and only when you enable all of the
above. Full walkthrough, security model, and troubleshooting:
[`docs/server-settings.md`](docs/server-settings.md).

## Co-located all-in-one dashboard (opt-in)

By default this stack forwards to a **separate** dashboard host. But
you can also run the dashboard **on this same host**, next to the game
server + companion — the `dashboard` + `cloudflared` services are
included but **off by default** behind a Compose profile.

Use this when the dashboard was hosted somewhere with a poor upload
pipe (a home server / Unraid box) and you want distant players to reach
a **datacenter VPS** directly instead. Two things improve:

- **Players hit the VPS, not your home uplink** — page loads + live
  SSE terminate in the datacenter.
- **The companion→dashboard hop becomes localhost** — no Cloudflare
  round-trip on either the slow `/ingest` lane or the fast
  `/ingest/vitals` lane. (Today the companion ships its telemetry out
  to the dashboard host and back, even when they're the same box.)

```
┌─────────────── your VPS (one host) ────────────────┐
│  enshrouded-server ──UDP──► players                │
│  beacon ── http://dashboard:8000 (localhost) ──┐   │
│  dashboard :8000  ◄─────────────────────────────┘  │
│   └─ SQLite /data/dashboard.db                     │
│  cloudflared ──outbound tunnel──► Cloudflare edge   │
└──────────────────────────▲─────────────────────────┘
                           │  your.domain (same hostname as before)
                     Friends (browse + sign in + SSE)
```

The public hostname **doesn't change**, so the move is invisible to
friends. Enable with:

```bash
# after migrating your DB + filling in the co-located .env vars:
docker compose --profile dashboard up -d
```

The cloudflared tunnel is **outbound** (it dials Cloudflare's edge), so
**no inbound port is opened on the VPS** — same security posture as a
home deployment.

**This is a system-of-record move** (your SQLite DB carries all player
/character history). Don't wing it — the full walkthrough covers
migrating the DB with zero data loss, the Cloudflare tunnel setup, the
companion repoint, backups (VPS → home via rsync), and a <5-minute
rollback:

→ [`docs/co-located-dashboard.md`](docs/co-located-dashboard.md)

## What this stack does NOT bundle

- **The dashboard host** — *by default*. This stack forwards data
  outbound to a separate dashboard. If you'd rather run the dashboard
  on this host too, see [Co-located all-in-one dashboard](#co-located-all-in-one-dashboard-opt-in)
  above (opt-in profile). See [`enshrouded-beacon`][source] for the
  dashboard's standalone deployment recipe.
- **Cloudflare tunnel or reverse proxy** — *for the game server*. The
  opt-in `cloudflared` service above is for the **co-located dashboard
  only**. If you want to expose this stack's *game server* publicly
  under a domain name rather than its raw IP, add your own proxy — most
  fellowship setups don't need this (game clients connect via IP).
- **Save-game backups.** Manual recipe above; not automated
  by default. Pull in `restic` or `borg` for off-host
  backups if your home server doesn't already do that.
- **Multi-server-on-one-host.** Running 2+ Enshrouded
  dedicated servers on the same Ubuntu host with one beacon
  each is possible but needs port-disambiguation +
  service-name uniqueness; not packaged here. Open an issue
  if you want this.

## How this fits in the multi-repo split

Source code, schemas, RE work, catalogs, tests — all live in
the source monorepo:

  **https://github.com/spacetoast1738/enshrouded-beacon**

This deploy repo only contains YAML + Markdown + `.env.example`
+ LICENSE. **No Python**. The companion-Beacon image is built
by the source repo's CI; we just reference it by tag.

Wire-format compatibility note: this stack's `beacon` image
requires a dashboard running [enshrouded-beacon][source]
**0.43.0+** (for the wire log surface) and 0.42.0+ for the
basic dedicated-server ingest path. The optional
[server-settings editing](docs/server-settings.md) +
host-utilisation features require dashboard + beacon **0.90.0+**.
Newer dashboards are backward-compat with older beacons
(additive-only fields).

## Credits

- Wine-based dedicated-server image:
  [`sknnr/enshrouded-dedicated-server`][sknnr] —
  Keen Software House's Enshrouded server running under
  Wine on Linux. MIT licensed.
- Auto-update: [`containrrr/watchtower`][wt] — Apache 2.0.
- Companion-Beacon: [enshrouded-beacon][source] — MIT.

[wt]: https://github.com/containrrr/watchtower

## License

MIT. See [`LICENSE`](LICENSE).
