# Diagnostic bundles

**TL;DR**: optional, opt-in (off by default), lets the dashboard
maintainer pull raw save files from your beacon for reverse-
engineering work. Set `ESB_ALLOW_DIAGNOSTIC_BUNDLES=1` in your
`.env` to enable. Skip this doc entirely if you don't want it on.

## What this is

The dashboard maintainer (the person operating
[enshrouded-beacon][source] — your dashboard host) can sometimes
benefit from access to **raw save files** from operator-deployed
servers like yours. Reasons:

- **Finding new fields** in Enshrouded's BDB1 save format.
  The structured wire payload your beacon posts on every
  `/ingest` is aggregated + redacted by design (passwords scrubbed,
  per-character data stripped). RE work needs the actual bytes.
- **Catalog mining**. The Enshrouded server's install dir
  contains `.dat` files with item / knowledge / craftable
  catalogs. New game versions add entries; pulling sample
  `.dat` files from operators on different game versions helps
  the dashboard learn about new items without the maintainer
  having to wait for the next sknnr image update on their
  own test box.
- **Per-world debugging**. If a specific world is producing
  weird metrics ("my codex count shows 10 less than my
  friend's after the same activity"), having the actual save
  file makes the diagnosis 100× faster than guessing from the
  wire payload alone.

[source]: https://github.com/spacetoast1738/enshrouded-beacon

## How it works (0.44.0+)

**Dashboard-initiated, beacon-fulfills on its next poll**:

1. The dashboard maintainer opens `/admin → Dedicated servers
   → your server row` and clicks **"Request bundle"**. They
   type an optional free-text note describing the gameplay
   moment ("made a beacon, started raining", "altar
   activated, level 4→5"). The note gets frozen into the
   bundle's `manifest.json` at request time so a corpus diff
   later still shows the original label.
2. The dashboard inserts a row in its
   `dedicated_server_bundle_requests` table with `status='pending'`
   and a 1-hour TTL (so a stale request doesn't haunt the
   queue forever).
3. Your beacon's existing `/ingest` POST loop now follows up
   with a quick `GET /ingest/tasks` call after each successful
   `/ingest`. The dashboard returns at most one task.
4. **If `ESB_ALLOW_DIAGNOSTIC_BUNDLES=1`** is set on your
   container, the beacon builds a gzipped tarball (described
   below) and POSTs it to the dashboard's
   `/admin/diagnostic-bundles/upload` endpoint.
5. **If `ESB_ALLOW_DIAGNOSTIC_BUNDLES=0`** (default, or unset),
   the beacon responds with `status=refused`. The dashboard
   admin sees "operator opt-out" on the request row instead
   of a stuck `pending` state. No data leaves your beacon.

End-to-end takes ≤ 30 seconds when opted in. The beacon's
existing 30-second poll cadence is the only latency.

## What's in a bundle (when enabled)

A bundle is a `.tar.gz` file at most 10 MB total:

```
diagnostic-bundle-r<request_id>-<timestamp>.tar.gz
├── manifest.json              ← admin's note (frozen), beacon
│                                version, image SHA, file paths,
│                                install-dat sha256 list
├── save/
│   ├── <world_id>-<N>         ← active save slot (KSC1 binary)
│   ├── <world_id>-index       ← rotation slot index
│   └── enshrouded_server.json ← config — passwords + tokens
│                                regex-scrubbed (3 passes)
├── logs/
│   └── enshrouded_server.log  ← last 1 MB of log
└── install/                   ← ONLY when "Include install .dat"
    ├── DAT_MANIFEST.json      ← sha256 + size per file
    └── <selected .dat files>  ← per allow-list: knowledge.dat,
                                  craftables.dat, items.dat,
                                  *catalog*.dat

Available since 0.44.4: the deploy compose now shares the
Steam install dir between the dedicated-server container and
the beacon (read-only on the beacon side) via the
`enshrouded-install` named volume. If you're on a deploy
that predates 0.44.4 OR you stripped the named volume from
your compose, the install section of the bundle is empty
and the manifest reports `install_dat_count: 0`. Pure
diagnostic — the bundle still ships, just without the
catalog `.dat` files.
```

**What's REDACTED from `enshrouded_server.json` before it ships**:
- Anything matching field-name regex `/(password|adminPassword|
  authToken|secret|key|token)/i` is replaced with
  `***redacted***`.
- Everything else (server name, slot count, port numbers) is
  preserved.

**What's NOT redacted**:
- **Save files**. That's the point of the feature — operator
  opted in for RE work. Save files contain player names,
  base positions, quest progress, codex contents, etc.
  By design.
- **Server logs**. The Enshrouded dedicated server's log
  contains player join/leave events with names. Last 1 MB
  by default, which on a quiet server is roughly the last
  several days of logs.

**What's NEVER in a bundle**:
- Your `ESB_TOKEN` (the bundle ships AS itself; the bearer
  header carries the token but never lands in the tarball).
- Files outside the explicit list above. We don't tar up
  the whole save dir or the whole install dir — only the
  files enumerated above.
- The actual game-server binary, Wine prefix, Steam cache
  files, etc.

## Read-safety: half-written saves

A naive "tar the live save file" would catch a half-written
file mid-flush. The beacon avoids this by only including the
newest rotation slot whose mtime is **> 5 seconds old**. If no
slot satisfies that gate at the moment a bundle is requested,
the manifest reports `save_active_slot: null` and the bundle
ships without a save (config + log + install only). The next
request typically succeeds because autosaves are infrequent
(default: every 5 min).

## Storage on the dashboard

Bundles are kept on the dashboard host at
`<DATABASE_PATH>/../diagnostic-bundles/`, named
`s<server_id>-r<request_id>-<timestamp>.tar.gz`.

**Retention**: 10 bundles per server (oldest pruned on each
upload). The dashboard admin can also manually delete any
bundle via the admin UI.

## What if I never want to enable this?

Leave `ESB_ALLOW_DIAGNOSTIC_BUNDLES` unset (or set to `0`),
which is the default. If the dashboard admin clicks "Request
bundle" anyway, the beacon politely refuses + the admin sees
"operator opt-out" on the request row. No data leaves your
beacon. **The routine `/ingest` wire payload continues
exactly as before** — only the new file-pull workflow is
gated by this flag.

To revoke after enabling: set `ESB_ALLOW_DIAGNOSTIC_BUNDLES=0`
in `.env`, then `docker compose up -d` to recreate the
beacon container with the new env. Existing bundles on the
dashboard side are not auto-deleted; ask the maintainer to
delete them if you want to.

## Verifying it works

After setting `ESB_ALLOW_DIAGNOSTIC_BUNDLES=1` in `.env` and
`docker compose up -d beacon`:

1. Ask the dashboard admin to issue a test bundle request.
2. Watch the beacon log via `docker compose logs -f beacon`
   — look for:
   ```
   built diagnostic bundle: diagnostic-bundle-rN-YYYYMMDD-HHMMSS.tar.gz
   diagnostic bundle uploaded: request_id=N, size=<X> B
   ```
3. The admin should see the bundle row in `/admin →
   Dedicated servers → wire log` (or the dedicated bundles
   list) within seconds.

## Threat model summary

- **Token compromise**: Your `ESB_TOKEN` is the only auth
  credential. Same threat model as before — if the token
  leaks, an attacker could spoof your beacon's heartbeats.
  Adding bundles to the equation expands the leak to "an
  attacker could request your save files were the dashboard
  to send a task, AND your `ESB_ALLOW_DIAGNOSTIC_BUNDLES=1`."
  Two layers of consent there.
- **Dashboard-side bundles**: stored on the dashboard
  maintainer's host. Same trust boundary as the wire
  payloads already in their DB.
- **Tarball in transit**: TLS terminated at the dashboard
  host's reverse proxy (Cloudflare tunnel, Caddy, nginx,
  etc.). Standard HTTPS guarantees.

If any of this feels uncomfortable, leave the flag off.
The dashboard's routine analytics — codex graphs, save-tick
telemetry, player-base POIs on the map — continue working
without bundles ever flowing.
