# Troubleshooting

Symptom → likely cause → fix. Most issues fall into one of
these five buckets.

## 1. Beacon container restarts in a loop

**Symptom**: `docker compose ps` shows `enshrouded-beacon`
with `Restarting (1) X seconds ago` over and over.
`docker compose logs beacon` shows
`POST /ingest rejected with 401` or
`POST failed: ConnectError: ...`.

**Likely causes**:

a. **Token mismatch.** The `ESB_TOKEN` in `.env` doesn't
   match the token issued from `/admin → Dedicated servers`.
   - Fix: re-issue the token in `/admin` (it's shown ONCE
     per issue), paste into `.env`, `docker compose
     restart beacon`.

b. **Dashboard URL wrong / unreachable.** `DASHBOARD_URL`
   typo, http vs https, port mismatch.
   - Fix: from inside the beacon container,
     `docker compose exec beacon python -c
     "import httpx; print(httpx.get('${DASHBOARD_URL}/api/players').status_code)"`
     should return 401 or 403 (auth failure is FINE here —
     it means the URL works). 200 / 0-byte means the
     URL is wrong.

c. **TLS / cert issue.** Dashboard's certificate is self-
   signed or expired.
   - Fix: use a real cert (Cloudflare tunnel handles this
     automatically). For dev, you can mount a CA bundle
     into the beacon container but we deliberately don't
     ship a flag for trust-bypass.

d. **Pinned dashboard version too old.** The beacon image
   ships features the dashboard doesn't recognise. Older
   dashboards silently accept; very-old might 422.
   - Fix: upgrade the dashboard host to 0.43.0+. The
     beacon's `client_version` field on the wire matches the
     image tag; admin can see this in the wire log.

## 2. Server container fails to start

**Symptom**: `docker compose ps` shows
`enshrouded-server` exited / restarting. Logs from
`docker compose logs enshrouded-server` show SteamCMD
errors, Wine prefix corruption, or "permission denied"
on `/home/steam/enshrouded`.

**Likely causes**:

a. **SteamCMD download failure.** Steam might be down or
   throttling; first-run downloads are ~3 GB.
   - Fix: retry `docker compose up -d`. SteamCMD resumes
     partial downloads. If it persistently fails, check the
     community image's [troubleshooting docs][sknnr-tshoot].

b. **Wine prefix corruption.** Mid-write crash, disk full,
   or container killed during initial Wine setup.
   - Fix: stop the stack, delete `data/enshrouded/.wine*`
     (the Wine prefix lives in the image's home dir; check
     the community image's docs for the exact path), `up
     -d` again. The Wine prefix re-creates cleanly.

c. **Disk full.** Save dir + Wine prefix + SteamCMD cache
   = ~5 GB minimum, plus world growth.
   - Fix: `df -h /` and clean up. Old Docker images are a
     common culprit: `docker image prune`.

d. **Permission denied on bind mount.** The host's
   `./data/enshrouded` directory has wrong ownership.
   Common signature in the logs:
   ```
   INFO: Server config not present, copying example
   cp: cannot create regular file
       '/home/steam/enshrouded/enshrouded_server.json':
       Permission denied
   ```
   followed by the container exiting with code 1 +
   restarting (it'll re-download the 8 GB Steam payload on
   every cycle, so stop the stack quickly).
   - Fix: the `sknnr/enshrouded-dedicated-server` image
     runs SteamCMD as uid **10001** (NOT 1000 — confirmed
     by `docker run --rm --entrypoint id
     sknnr/enshrouded-dedicated-server:latest`). On first
     `docker compose up`, Docker auto-creates
     `./data/enshrouded` owned by `root:root`, which
     uid 10001 can't write to.
     ```bash
     docker compose down
     sudo chown -R 10001:10001 ./data/enshrouded \
                               ./data/beacon-state
     docker compose up -d
     ```
   - The beacon-state dir gets the same treatment so the
     beacon container can write its tail-pos sidecar.
   - This is a first-time-setup issue only; once the dirs
     are owned correctly, subsequent `up -d` cycles work
     fine.

## 3. Dashboard shows the server but no save-tick events

**Symptom**: `/admin → Dedicated servers` shows the row with
`last_seen` advancing (so the beacon IS posting), but the
`last_save_tick_ts` column stays empty / never advances.
Wire log shows payloads arriving but with
`client_state.save_tick_compression_us == {}`.

**Likely causes**:

a. **`ESB_LOG_PATH` mount issue.** The beacon can't find
   `enshrouded_server.log` at the configured path.
   - Fix: `docker compose exec beacon ls -la /logs/` should
     show `enshrouded_server.log`. If it's empty, the
     community image is writing logs elsewhere. Find the
     real log path inside the `enshrouded-server` container
     (`docker compose exec enshrouded-server find / -name
     enshrouded_server.log 2>/dev/null`), update the
     `volumes:` block in `docker-compose.yml` accordingly.

b. **Log file is fresh / no save events yet.** The
   dedicated server saves every 5 minutes by default. If
   the server just booted and no players have joined / done
   anything, the log won't have `[savedata]` lines yet.
   - Fix: wait at least 6 minutes after server boot. The
     first save tick should appear in the log + then in the
     dashboard.

c. **Log file rotated / truncated mid-read.** If the
   community image rotates logs aggressively, the beacon's
   saved position may end up past EOF.
   - Fix: `docker compose exec beacon rm /state/tail-pos.json`
     forces a full re-read on next poll. Cheap (one
     ~70 KB log).

## 4. Players can't connect (server invisible / connection refused)

**Symptom**: friends say the server doesn't show up in the
in-game server browser, or attempts to direct-connect to
`<host-public-ip>:15637` time out.

**Likely causes**:

a. **UDP ports not forwarded** through the router. The host
   has the ports open (`ufw status` confirms) but the
   router's NAT isn't routing external UDP to the host's
   LAN IP.
   - Fix: see [`port-forwarding.md`](port-forwarding.md)
     for the full recipe.

b. **`ufw` blocking** the ports.
   - Fix: `sudo ufw allow 15636:15638/udp`, `sudo ufw
     reload`.

c. **Server not actually listening yet.** First-boot
   SteamCMD download can take 10+ minutes; the server isn't
   accepting connections until it logs "ready" (community-
   image-specific wording).
   - Fix: `docker compose logs -f enshrouded-server` and
     wait for the ready-state log line.

d. **CGNAT** (carrier-grade NAT, common on starter
   fibre / mobile plans) means you don't have a real public
   IPv4 and inbound connections never reach the host.
   - Fix: ask ISP for a public IP, or move the stack to a
     VPS, or use a STUN-less tunnel (rare for game UDP
     servers — typically need a paid solution).

## 5. "AOB not matched" / signature errors in beacon logs

**Symptom**: `docker compose logs beacon` shows
`frida hook reported: AOB not matched — game patched?
signature stale?` or similar.

**This applies to the PLAYER Beacon, not the server-Beacon.**
The server-Beacon doesn't use Frida (it's a pure file-tail
consumer with no memory hooks). If you see this in the
server-Beacon logs, file an issue — it shouldn't happen.

If you see it in a player Beacon (separate from this stack),
the fix is to upgrade the player Beacon to the current
release, which usually contains an updated signature.

## When all else fails

1. **Collect logs**:
   ```bash
   docker compose logs --tail=200 enshrouded-server > server.log
   docker compose logs --tail=200 beacon > beacon.log
   docker compose ps > containers.txt
   docker compose config > resolved-compose.yml
   ```
2. **File a GitHub issue** at
   [enshrouded-beacon-server-stack/issues][issues]. Paste
   the four files above (scrub `ESB_TOKEN` from
   `resolved-compose.yml`!) plus your `.env.example` for
   reference.
3. For dashboard-side issues, file at
   [enshrouded-beacon/issues][source-issues]. The wire log
   on the admin dashboard is the most useful diagnostic —
   open the failing payload and screenshot the full JSON.

[sknnr-tshoot]: https://github.com/jsknnr/enshrouded-server#troubleshooting
[issues]: https://github.com/spacetoast1738/enshrouded-beacon-server-stack/issues
[source-issues]: https://github.com/spacetoast1738/enshrouded-beacon/issues
