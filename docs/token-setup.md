# Token setup

The beacon container needs a plaintext **ingest token** to
authenticate POSTs to your dashboard. Issue it once from the
dashboard's admin UI, paste it into `.env`, and you're done.

## Issuing a token

1. Open your dashboard in a browser (you're already an admin
   on it — this stack is for forwarding data to a host you
   already operate).
2. Navigate to **`/admin → Dedicated servers`**.
3. Click **"Register a new dedicated server"**.
4. Fill in a display name. Recommended: match your
   `SERVER_NAME` env var (e.g. "My Fellowship Server") so the
   admin UI's row matches the in-game server name. The two
   don't have to match — the beacon image's
   `ESB_DISPLAY_NAME` env defaults to `SERVER_NAME` for
   convenience.
5. Click **"Create"**. The dashboard shows the **plaintext
   token ONCE**. It looks like a 43-character
   base64-url-safe string (e.g. `aB3xYzKjL...`).
6. **Copy it immediately.** The dashboard only stores the
   sha256 hash; you can't recover the plaintext later.

## Pasting into `.env`

```bash
cd /opt/enshrouded-beacon-server-stack    # or wherever you cloned
$EDITOR .env
```

Find the `ESB_TOKEN=` line and paste the token after the `=`
with NO quotes, NO trailing whitespace:

```env
ESB_TOKEN=aB3xYzKjL...
```

Then start (or restart) the stack:

```bash
docker compose up -d                # first time
docker compose restart beacon       # if already running
```

## Verifying the token works

Within 30 seconds of restart, the beacon's first POST should
hit `/ingest` on your dashboard. Verify:

- **Beacon logs**: `docker compose logs beacon | head -20`
  shows the startup config line + the first
  `posted world: id=...` log line.
- **Dashboard admin row**: `/admin → Dedicated servers` shows
  this server with a recent `last_seen` (under a minute ago).
- **Wire log**: `/admin → Dedicated-server wire log` shows
  the first payload arriving with a 204 status.

## Token rotation

If you ever need to rotate the token (suspected leak, friend
left the team, etc.):

1. Open `/admin → Dedicated servers`.
2. Click **"Reissue"** next to the server row. The dashboard
   shows a NEW plaintext token; the old one is immediately
   invalidated.
3. Paste the new token into `.env`, replacing the old value.
4. `docker compose restart beacon`.

The dashboard preserves all historical state for that server
(world, POIs, save-tick history) across the rotation — only
the auth credential changes.

## Revoking entirely

If you want to stop a beacon from being able to post (e.g.
shutting down a stack permanently):

1. Open `/admin → Dedicated servers`.
2. Click **"Revoke"** next to the server row. The token is
   swapped for a `revoked:...` sentinel value; subsequent
   POSTs return 401.
3. The row + its historical state are preserved (audit
   trail). Click **"Delete"** if you really want the row
   gone — note `dedicated_server_pois`, `dedicated_server_
   ingest_log`, and any related rows CASCADE-delete with it.

## Security notes

- The token is the equivalent of a **password** for the beacon
  posting on this server's behalf. Treat it accordingly:
  - Never commit `.env` (it's in `.gitignore` by default).
  - Restrict file permissions:
    ```bash
    chmod 600 .env
    chown $USER:$USER .env
    ```
  - Avoid pasting the token into chat / issue trackers.
- The dashboard scrubs `Bearer <token>` / `ingest_token=`
  patterns from the wire log before persisting (0.43.0+), so
  even if you accidentally include the token inside a server-
  Beacon log line, it won't leak to viewers.
- Tokens are per-server, NOT per-user. A token compromise
  only affects this dedicated server's data, not the
  dashboard itself or other servers' tokens.
