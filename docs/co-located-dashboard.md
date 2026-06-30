# Co-located all-in-one dashboard — migration walkthrough

Move the **dashboard** off its current host (e.g. a home Unraid box
behind a house Cloudflare tunnel) and onto **this VPS**, next to the
game server + companion. Distant players then reach a datacenter
instead of your home uplink, and the companion→dashboard hop collapses
to localhost.

This is a **system-of-record move** — your SQLite `dashboard.db`
carries every player/character's history, sessions, events, settings
audit, and codex unlocks. The steps below move it with **zero data
loss** and keep a **<5-minute rollback** the whole way through. Read it
once end-to-end before starting.

> Conventions: **[VPS]** = run on the game-server VPS. **[HOME]** = run
> on the box currently hosting the dashboard. **[CF]** = Cloudflare
> dashboard (web UI). Paths/hostnames below use this project's
> example values — substitute your own.

---

## What changes (and what doesn't)

| | Before | After |
|---|---|---|
| Dashboard host | home box | this VPS |
| Public hostname | `beacon.spencer-net.com` | **unchanged** |
| Friends' Beacon config | same URL | **unchanged** |
| Companion → dashboard | over Cloudflare (out to home + back) | **localhost** |
| TLS | home Cloudflare tunnel | new VPS Cloudflare tunnel |
| DB location | home `/data/dashboard.db` | VPS `./data/dashboard/dashboard.db` |
| Inbound VPS ports | none (tunnel is outbound) | **still none** |

The cutover is invisible to friends if the tunnel + DNS are moved
cleanly. The wire format is unchanged; old Beacons keep posting.

---

## Prerequisites

1. **Docker + Compose v2** on the VPS (the stack already needs this).
2. **GHCR access for the dashboard image.** The dashboard image
   (`ghcr.io/spacetoast1738/enshrouded-beacon`) may be private. If a
   later `docker compose pull` fails with `denied`/`unauthorized`, log
   the VPS docker daemon in once:
   ```bash
   # [VPS]
   echo <GHCR_PAT> | docker login ghcr.io -u spacetoast1738 --password-stdin
   ```
   (A read-`packages` PAT is enough.)
3. **A shared vitals secret** if you use the 0.106.0 direct-vitals
   overlay path — generate once and use the SAME value on both the
   dashboard and the companion:
   ```bash
   openssl rand -hex 32
   ```
4. **Your Cloudflare account** (the one owning the public hostname's
   zone) for the Zero Trust tunnel step.
5. The **auth-email (Resend) values** your current dashboard uses
   (`RESEND_API_KEY`, `RESEND_FROM`) — copy them over so sign-in keeps
   working.

---

## Phase 0 — Pre-flight (no downtime)

**0.1 [VPS] Confirm headroom.** The dashboard is a single FastAPI
worker + SQLite — budget ~150-300 MB RSS idle, a fraction of a core for
a small friend group. The real question is whether the *game server's*
spikes leave that room:

```bash
# [VPS]
free -m ; nproc ; df -h / ; docker stats --no-stream
```

If RAM is tight, add a `mem_limit:` to the `dashboard` service in
`docker-compose.yml` so a dashboard spike can't OOM-kill the game
server. (Ask and we'll wire it in.)

**0.2 [VPS] Fill in the co-located `.env`.** In the stack's `.env`
(see `.env.example` → "Co-located all-in-one dashboard"):

```ini
DASHBOARD_URL=http://dashboard:8000          # companion → local dashboard
PUBLIC_BASE_URL=https://beacon.spencer-net.com   # KEEP your hostname
CLOUDFLARED_TOKEN=                           # filled in Phase 4 (skip if you
                                             #   front it with an existing proxy)
RESEND_API_KEY=...                           # copy from old dashboard
RESEND_FROM=Enshrouded Beacon <noreply@spencer-net.com>
GITHUB_TOKEN=                                # optional
VITALS_TICKET_SECRET=<openssl rand -hex 32>  # if using direct vitals
```

> **Already running Caddy / Traefik / nginx on this host?** (e.g. for another
> service.) Skip the `cloudflared` service entirely — add the `dashboard` to
> your proxy's docker network and a vhost pointing at
> `enshrouded-dashboard:8000`, then in Phase 4 just add the DNS A record →
> your VPS IP and let your proxy issue the cert. This is how the reference
> deployment is actually run (Caddy + an existing `*.spencer-net.com` setup).

> Leave `CLOUDFLARED_TOKEN` blank for now — you'll get it in Phase 4.
> Don't start the `dashboard` profile until the DB is in place (Phase 3).

**0.3 [VPS] Create the data dir.**
```bash
# [VPS]  (from the stack directory)
mkdir -p data/dashboard
```

---

## Phase 1 — Dry-run on the VPS (still no cutover)

Prove the image runs here with an **empty** DB on a throwaway port, no
tunnel:

```bash
# [VPS]
docker run --rm -p 127.0.0.1:8001:8000 \
  -e DATABASE_PATH=/data/dashboard.db -e COOKIE_SECURE=0 \
  -v "$(pwd)/data/dryrun:/data" \
  ghcr.io/spacetoast1738/enshrouded-beacon:latest &
sleep 8
curl -s localhost:8001/healthz && echo
```

Expect `/healthz` → 200 and the logs to show migrations applying
cleanly to the fresh DB before uvicorn binds. Then stop it and clean
up:

```bash
# [VPS]
docker stop $(docker ps -q --filter ancestor=ghcr.io/spacetoast1738/enshrouded-beacon:latest)
rm -rf data/dryrun
```

---

## Phase 2 — Migrate the SQLite DB (the careful part)

SQLite in WAL mode means `dashboard.db` alone is **not** a consistent
snapshot — uncommitted pages live in `dashboard.db-wal`. The safe move
is to stop the writer, fold the WAL in, verify, copy.

**2.1 [HOME] Quiesce writes.** Stop the dashboard so nothing holds the
DB. Beacons keep running; their POSTs fail/retry during the window
(friends see a brief "offline", no data lost — the wire is additive and
they reconnect).

```bash
# [HOME]  (Unraid: container name is enshrouded-beacon)
docker stop enshrouded-beacon
```

**2.2 [HOME] Checkpoint + integrity-check.** With the container
stopped:

```bash
# [HOME]  in the dir holding dashboard.db (e.g. /mnt/user/appdata/enshrouded-beacon)
sqlite3 dashboard.db 'PRAGMA wal_checkpoint(TRUNCATE);'
sqlite3 dashboard.db 'PRAGMA integrity_check;'     # MUST print: ok
sha256sum dashboard.db ; ls -l dashboard.db
```

After a `TRUNCATE` checkpoint the `-wal`/`-shm` files are empty and the
single `dashboard.db` is the whole story. If `sqlite3` isn't installed
on the home box, instead copy **all three** files (`dashboard.db`,
`dashboard.db-wal`, `dashboard.db-shm`) and let the new container
replay the WAL — but the checkpoint path is cleaner.

**2.3 Transfer to the VPS.** Don't paste it through a terminal — use
`scp` (large binary). From a machine that can reach both, or push from
home:

```bash
# [HOME]
scp -i ~/.ssh/enshrouded_re_ed25519 dashboard.db ubuntu@57.128.191.219:/tmp/dashboard.db
```

**2.4 [VPS] Install + re-verify.** Match the container's `app` user
(uid 1000); the entrypoint re-chowns `/data` anyway, but matching
up front avoids surprises:

```bash
# [VPS]  (from the stack directory)
sudo install -o 1000 -g 1000 -m 644 /tmp/dashboard.db data/dashboard/dashboard.db
sha256sum data/dashboard/dashboard.db          # MUST match the home checksum
sqlite3 data/dashboard/dashboard.db 'PRAGMA integrity_check;'   # ok
rm /tmp/dashboard.db
```

If `integrity_check` doesn't print `ok` or the checksums differ:
**stop** — re-transfer. Your home DB is untouched; nothing is lost.

---

## Phase 3 — Bring the dashboard up on the VPS (still no public traffic)

```bash
# [VPS]
docker compose --profile dashboard up -d dashboard
docker compose logs -f dashboard     # migrations idempotent (already current), uvicorn binds :8000
curl -s localhost:8000/healthz && echo   # 200
```

Sanity-check the data made it. From your laptop, SSH-tunnel in and open
it in a browser:

```bash
# [your laptop]
ssh -i ~/.ssh/enshrouded_re_ed25519 -L 8000:localhost:8000 ubuntu@57.128.191.219
# then browse http://localhost:8000 — log in, confirm players /
# characters / history / server pages are all present.
```

Don't start `cloudflared` yet — verify the data first.

---

## Phase 4 — Cloudflare tunnel + DNS cutover (the visible flip)

The public hostname must now resolve to the VPS dashboard. Use a **new
tunnel on the VPS** (outbound — no inbound port exposed).

**4.1 [CF] Create a VPS tunnel.** Cloudflare Zero Trust → Networks →
Tunnels → *Create a tunnel* → Cloudflared → name it (e.g.
`enshrouded-vps`). Copy the **tunnel token** it shows.

**4.2 [CF] Add the public hostname.** On that tunnel, add a Public
Hostname:
- **Subdomain/domain:** `beacon.spencer-net.com` (your existing one)
- **Service:** `HTTP` → `dashboard:8000`

(Because `cloudflared` runs in this compose project, it resolves the
`dashboard` service by name over the compose network.)

**4.3 [VPS] Start cloudflared.** Put the token in `.env`
(`CLOUDFLARED_TOKEN=...`) and bring up the full profile:

```bash
# [VPS]
docker compose --profile dashboard up -d        # starts dashboard + cloudflared
docker compose logs -f cloudflared              # look for "Registered tunnel connection"
```

**4.4 Flip + verify externally.** Cloudflare lets the old and new
tunnels coexist briefly. Once the VPS tunnel is healthy and (if needed)
you've repointed the `beacon.spencer-net.com` route to it:

```bash
# [anywhere external]
curl -sI https://beacon.spencer-net.com/healthz
```

Confirm: sign-in works (cookie is `Secure` over TLS), the dashboard
loads, SSE live-updates flow (watch a value tick), a friend's Beacon
row is green. DNS/tunnel propagation is seconds-to-low-minutes.

SSE note: the app sets `X-Accel-Buffering: no` + a 15 s heartbeat
itself, and Cloudflare's edge behaviour is unchanged, so streaming
keeps working through the new tunnel with no extra config.

---

## Phase 5 — Repoint the companion to localhost

Now that the dashboard is local, stop sending companion telemetry out
to Cloudflare and back. In `.env` you already set
`DASHBOARD_URL=http://dashboard:8000` (Phase 0.2). Re-create the
companion so it picks that up:

```bash
# [VPS]
docker compose up -d beacon
docker compose logs -f beacon       # first POST should 2xx against the local dashboard
```

This single env var drives **all three** companion endpoints
(`/ingest`, `/ingest/vitals`, `/ingest/tasks`), so one change collapses
the entire companion→hub path to localhost. The companion posts over
plain HTTP on the compose network — fine, it never leaves the box.
(Keep the dashboard's `COOKIE_SECURE=1`; that governs *browser* cookies
over the public tunnel, not the localhost ingest path.)

If you use the **0.106.0 direct-vitals overlay path**, make sure the
companion's `ESB_VITALS_TICKET_SECRET` equals the dashboard's
`VITALS_TICKET_SECRET`. (See the note at the bottom — co-location
shrinks but doesn't remove the direct path's value; keep it, measure
later.)

---

## Phase 6 — Decommission home (after a grace period)

Leave the home dashboard **stopped but not deleted** for 24-48 h as a
hot rollback target. After you're satisfied:

```bash
# [HOME]
docker rm enshrouded-beacon         # remove the stopped container
# archive the pre-migration DB copy as a cold backup; keep it.
```

Then in Cloudflare, remove the **old** home tunnel's
`beacon.spencer-net.com` public-hostname route (so only the VPS tunnel
serves it).

---

## Backups (VPS nightly + pull home)

The home box's array/parity was your backup tier; recreate that. Best
of both: nightly consistent snapshot **on the VPS**, pulled **home** by
rsync so the home box becomes the backup tier instead of the host.

**VPS — nightly `.backup`** (consistent even with the container
writing; don't `cp` the live `.db` alone):

```bash
# [VPS]  /etc/cron.daily/backup-dashboard   (chmod +x)
#!/bin/sh
set -eu
STAMP=$(date +%F)
OUT=/opt/enshrouded-beacon-server-stack/data/dashboard-backups
mkdir -p "$OUT"
docker exec enshrouded-dashboard \
  sqlite3 /data/dashboard.db ".backup /data/backup-$STAMP.db"
mv /opt/enshrouded-beacon-server-stack/data/dashboard/backup-$STAMP.db "$OUT/"
gzip -f "$OUT/backup-$STAMP.db"
find "$OUT" -name 'backup-*.db.gz' -mtime +14 -delete   # keep 14 days
```

**Home — pull the backups** (turns the home box into the offsite tier;
survives total VPS loss):

```bash
# [HOME]  nightly cron
rsync -az -e 'ssh -i ~/.ssh/enshrouded_re_ed25519' \
  ubuntu@57.128.191.219:/opt/enshrouded-beacon-server-stack/data/dashboard-backups/ \
  /mnt/user/backups/enshrouded-beacon/
```

> **⚠ Mind the timezones when scheduling.** The snapshot cron runs in the VPS's
> timezone and the pull cron runs in the home box's — if those differ (e.g. VPS
> on UTC, home on BST/GMT) a "1-hour gap" in wall-clock minutes can be **zero**
> in absolute time, so the pull races a half-written snapshot. Put the snapshot
> at an early *UTC* hour (e.g. `0 2 * * *`) so the home pull is always hours
> later regardless of season.
>
> **Harden the pull key.** Don't authorize the home box's key for full shell on
> the VPS. Restrict it to rsync-pull-only in the VPS's `~/.ssh/authorized_keys`:
> `restrict,command="rrsync -ro /path/to/dashboard-backups" ssh-ed25519 AAAA... home-backup`
> (`rrsync` ships with rsync 3.2.4+). Then the home client points at the rrsync
> root — source becomes `ubuntu@<vps>:` (empty path), not the full dir path.

Keep the Phase 2 pre-migration DB copy as your day-0 cold backup.

---

## Rollback (fast, <5 min)

Because home is left up + idle through Phase 6:

1. **[CF]** Repoint `beacon.spencer-net.com` back to the **home** tunnel/hostname.
2. **[HOME]** `docker start enshrouded-beacon`.
3. **[VPS]** Set `DASHBOARD_URL` back to `https://beacon.spencer-net.com` in `.env`, `docker compose up -d beacon`, and `docker compose --profile dashboard stop` to idle the VPS dashboard.

**Data caveat:** any writes that landed on the VPS DB after cutover are
not on the home DB. If you roll back after the VPS has served live
traffic for a while, you lose that delta. Keep the grace window short,
or accept the gap (it's session/event history, not money).

---

## Risks at a glance

- **DB move** — mitigated by stop-writer + checkpoint + `integrity_check`
  on both ends + checksum match. Home DB stays untouched until you
  delete it.
- **Tunnel/DNS** — stage the VPS tunnel and `curl` the public
  `/healthz` *before* removing the home route; both can coexist.
- **VPS resource contention** — the new risk. Dashboard + weekly
  SQLite `VACUUM` + SSE fan-out now share the host with the game
  server. Phase 0 headroom check + optional `mem_limit`. The VACUUM is
  gated to a 7-day cadence and survives restarts, so it runs at most
  once/week.
- **Single point of failure** — game + dashboard now share one host. A
  VPS outage takes down both. Acceptable: when the game's down nobody's
  playing, so the dashboard being down matters less — but it's now a
  chosen trade-off, not a surprise.

---

## A note on the 0.106.0 direct-vitals path

The direct VPS→overlay vitals path (`ESB_VITALS_DIRECT_ENABLED`) was
built to bypass the *home upload pipe* for friends' overlays. Once the
dashboard is co-located, the companion→hub hop is already localhost, so
that rationale shrinks — but the direct path still gives overlays a
dedicated TLS socket rather than multiplexing through the hub's SSE.

**Keep it as-is through the migration.** After everything's settled,
A/B the overlay with the direct path on vs off (the 0.105.3 latency
diagnostics in `beacon.log` measure both). If hub-relayed
`/events/vitals` is now indistinguishable, you can simplify by turning
the direct server off later — but measure first, don't decide
pre-emptively. Until then keep `VITALS_TICKET_SECRET` equal on both
sides.
