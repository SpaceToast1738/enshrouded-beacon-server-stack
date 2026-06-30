# Edit server settings from the website (0.90.0+, opt-in)

By default the beacon is **read-only** — it forwards stats and never
touches your server's files. Starting with **0.90.0**, you can
optionally let admins on your dashboard **edit this server's
`gameSettings`** (difficulty multipliers, enemy/boss/XP factors,
weather, day-night cycle, tombstone mode, etc.) and **restart the
server** to apply them, all from the web UI.

It's **off by default** and opt-in at several layers. This page is
the full setup. Read it end-to-end before enabling — two of the
steps (`EXTERNAL_CONFIG` and the file `chown`) are easy to miss and
the feature **fails silently** without them.

## Requirements

- Dashboard **and** beacon image **0.90.0+** (Watchtower pulls the
  beacon image automatically; make sure your dashboard host is also
  on 0.90.0+).
- You're comfortable editing `docker-compose.yml` + `.env`.

## ⚠ The two gotchas, up front

1. **`EXTERNAL_CONFIG=1` is mandatory.** The community
   dedicated-server image (`sknnr/enshrouded-dedicated-server`)
   **regenerates `enshrouded_server.json` from environment variables
   on every container start** by default. So if the dashboard writes
   your gameSettings and then the server restarts, the image would
   **overwrite your edits** on boot — the change would appear to
   "revert itself." Setting `EXTERNAL_CONFIG=1` tells the image to
   **preserve** the existing config file instead of regenerating it.
   With it set, your `enshrouded_server.json` becomes the source of
   truth and dashboard edits survive restarts.

2. **The config file must be writable by the beacon's user.** The
   dedicated server runs as uid **10000**; the beacon runs as uid
   **10100**. For the beacon to write `enshrouded_server.json` you
   must `chown` it to 10100. The server only *reads* the file in
   `EXTERNAL_CONFIG` mode, so read-for-others (`644`) is enough for
   it.

## Setup — write settings (no restart)

This lets the dashboard write gameSettings; you restart the server
yourself when convenient.

1. **Seed a full config file** (if you don't already have one). With
   `EXTERNAL_CONFIG=1`, the image stops generating the file, so you
   must supply a complete `enshrouded_server.json`. Easiest path:
   boot once in the DEFAULT (non-external) mode so the image writes a
   baseline file, then switch to external mode:

   ```bash
   docker compose up -d enshrouded-server
   # wait for data/enshrouded/enshrouded_server.json to appear
   docker compose stop enshrouded-server
   ```

   Make sure the file has a `gameSettings` object and
   `"gameSettingsPreset": "Custom"` (the dashboard sets this
   automatically when it writes, but seeding it as Custom avoids the
   first edit being a surprise).

2. **Enable external config** in `.env`:

   ```ini
   EXTERNAL_CONFIG=1
   ```

3. **Flip the config mount to `:rw`** in `docker-compose.yml` — find
   the beacon service's config bind and change the suffix:

   ```yaml
   # before:
   - ./data/enshrouded/enshrouded_server.json:/config/enshrouded_server.json:ro
   # after:
   - ./data/enshrouded/enshrouded_server.json:/config/enshrouded_server.json:rw
   ```

4. **Make the file beacon-writable:**

   ```bash
   sudo chown 10100 data/enshrouded/enshrouded_server.json
   sudo chmod 644 data/enshrouded/enshrouded_server.json
   ```

5. **Allow writes** in `.env`:

   ```ini
   ESB_ALLOW_SETTINGS_WRITE=1
   ```

6. **Apply:**

   ```bash
   docker compose up -d
   ```

Now edit settings on the dashboard (`/server/{id}` → **Edit
settings**). The write lands in `enshrouded_server.json`; because
restart isn't enabled yet, the dashboard shows **"restart pending"**
and you apply it with `docker compose restart enshrouded-server` when
you're ready.

## Setup — also restart from the website

To let the dashboard restart the server for you (so edits apply
without an SSH session), add the hardened docker-socket-proxy.

7. **Uncomment the `docker-proxy` service** in `docker-compose.yml`
   (it's provided commented-out near the bottom). It exposes only
   `CONTAINERS=1` + `POST=1` of the docker API and mounts the docker
   socket read-only — enough to restart a container, nothing else.

8. **Enable restart** in `.env`:

   ```ini
   ESB_ALLOW_SERVER_RESTART=1
   ESB_SERVER_CONTAINER_NAME=enshrouded-server
   ```

9. `docker compose up -d`.

Now ticking **"Restart now"** in the dashboard's edit dialog writes
the config and restarts `enshrouded-server` automatically. Players
disconnect briefly (the server flushes its save on shutdown).

## How it looks on the dashboard

- `/server/{id}` → **Edit settings** (admin only) → change values →
  **Review** the diff → **Apply** (optionally with **Restart now**).
- Status updates live: `applied`, `failed`, or `refused`.
- The change also shows up in the server's settings-history (`⚙`
  badges) on the next telemetry poll.

## Security model

- The **save directory stays mounted read-only** (`/save:ro`). This
  feature only makes the single *config file* writable, and only when
  you opt in. The "beacon never modifies your save" guarantee is
  unchanged.
- The beacon **never touches the raw docker socket** — only the
  hardened proxy does, and only for container operations.
- **Residual:** docker-socket-proxy can't scope `POST` to a single
  container, so with the restart gate on, the beacon could restart
  (only restart — not exec/build/pull) any container the proxy sees.
  The default-off gate + the proxy's operation allow-list are the
  mitigations. If that residual matters to you, leave
  `ESB_ALLOW_SERVER_RESTART=0` and restart manually.
- Everything is admin-gated on the dashboard and validated twice
  (dashboard at request time, beacon at write time) — out-of-range
  or unknown settings are rejected, never written.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Dashboard shows **"refused"** | `ESB_ALLOW_SETTINGS_WRITE` not set | Set it to `1` in `.env`, `docker compose up -d` |
| Edits **revert after a restart** | `EXTERNAL_CONFIG` not set — image regenerated the config | Set `EXTERNAL_CONFIG=1`, recreate the server container |
| Beacon logs **"permission denied"** writing the config | File not owned by uid 10100 | `sudo chown 10100 data/enshrouded/enshrouded_server.json` |
| Beacon logs **"config not found"** | Config mount missing / wrong path | Confirm the `:rw` bind + `ESB_SERVER_CONFIG_PATH=/config/enshrouded_server.json` |
| Dashboard shows **"restart pending"** after apply | `ESB_ALLOW_SERVER_RESTART` off (this is expected) | Restart manually, or enable the restart path (steps 7–9) |
| Restart reports **"no container named …"** | `ESB_SERVER_CONTAINER_NAME` wrong | Match it to `container_name:` on the server service |
| Restart reports **proxy unreachable** | `docker-proxy` service still commented out | Uncomment it, `docker compose up -d` |

## Turning it back off

Set `ESB_ALLOW_SETTINGS_WRITE=0` (and `ESB_ALLOW_SERVER_RESTART=0`),
optionally revert the config mount to `:ro`, and `docker compose up
-d`. The beacon returns to pure read-only telemetry. You can leave
`EXTERNAL_CONFIG=1` if you prefer managing the config file by hand.
