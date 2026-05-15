# Upgrading

This stack defaults to **auto-updates via Watchtower**: every
5 minutes it polls `:latest` for both images and re-pulls
anything new. For most operators that's all you need to know.

## How auto-update works (default)

Watchtower runs as a sidecar container, mounted into the host's
Docker socket so it can pull new images + restart your
containers with the new image. Configured in
`docker-compose.yml`:

```yaml
watchtower:
  image: containrrr/watchtower:latest
  command: --interval 300 --cleanup
```

- `--interval 300` = poll every 5 minutes.
- `--cleanup` = remove old image layers after pulling new ones
  (saves disk over time).
- No `--label-enable` filter = updates ALL containers in this
  compose project (so both `enshrouded-server` and `beacon`).

When a new `:latest` image is pushed by the source repo's
CI (or by the community image's maintainer for
`sknnr/enshrouded-dedicated-server`), Watchtower:

1. Pulls the new image (~2-15 MB diff in practice).
2. Stops the affected container gracefully.
3. Re-creates the container with the new image, preserving
   volumes + env vars.
4. Starts it.

Total downtime per update: typically 3-10 seconds for the
beacon, 15-30 seconds for the dedicated server (cold-restart
the Wine process).

## Watching auto-updates land

```bash
docker compose logs -f watchtower
```

Look for lines like:
```
Found new ghcr.io/spacetoast1738/enshrouded-beacon-server:latest image
Stopping /enshrouded-beacon (...) with SIGTERM
Creating /enshrouded-beacon
```

## Manual updates

Sometimes you want to force-update without waiting up to 5
minutes for the next Watchtower poll:

```bash
docker compose pull              # pulls all images' :latest
docker compose up -d             # recreates any container
                                 # whose image changed
docker compose images            # confirm new digests
```

`docker compose up -d` is a no-op for containers whose images
didn't change.

## Pinning specific versions (production stability)

If you want exact reproducibility — same image version
across all servers, no surprise updates — pin image tags:

1. Open `docker-compose.yml`.
2. Replace `:latest` with a specific tag for each service:
   ```yaml
   image: ghcr.io/spacetoast1738/enshrouded-beacon-server:0.43.1
   ...
   image: sknnr/enshrouded-dedicated-server:0.7.0.5
   ```
   Find available beacon tags at
   [the source repo's releases][releases].
   Find available community-server tags at
   [Docker Hub][sknnr-tags].
3. Comment out (or remove) the entire `watchtower` service
   to disable auto-updates.
4. `docker compose up -d` to apply.

To upgrade later: edit the tag, run `docker compose pull &&
docker compose up -d`.

## Pinning by digest (maximum stability)

Tags can be moved by image authors (rare but happens during
emergency rebuilds). For build-bit-identical reproducibility,
pin by SHA256 digest:

```yaml
image: ghcr.io/spacetoast1738/enshrouded-beacon-server@sha256:abc123...
```

Find the current digest via:
```bash
docker compose images --quiet | xargs docker inspect \
  --format='{{index .RepoDigests 0}}'
```

## Rollback

**Watchtower-managed rollback**: not built in. If a `:latest`
push breaks your stack, manually pin to the previous version
tag using the steps above. Then file an issue against the
broken image's repo.

**Manual rollback** (pinned-tag mode):
1. Edit `docker-compose.yml`, revert the tag to the last
   known-good version.
2. `docker compose up -d`.

**Save-state rollback** is a separate concern — see the
backup recipe in [`../README.md`](../README.md#backups).
Restore the tarball, then start the stack.

## Game-version coordination

The community Wine image's version is tied to the Enshrouded
*server-binary* version (via SteamCMD). When Keen ships a
game update:

1. Players connecting with an old client get rejected by the
   newer server (or vice versa).
2. The community image usually publishes a new tag within
   hours of release.
3. Watchtower picks it up automatically (default).

If you're tag-pinning and want to upgrade after a game
update: check [the community image's releases][sknnr-releases]
for the new tag, bump in your compose file, and `docker
compose up -d`.

**The beacon image's release cadence is independent** of game
updates. The beacon only reads files; it doesn't talk to the
game server's running process. Beacon stays compatible across
game updates unless the SAVE FORMAT changes — rare, and would
warrant a coordinated beacon release.

[releases]: https://github.com/spacetoast1738/enshrouded-beacon/releases
[sknnr-tags]: https://hub.docker.com/r/sknnr/enshrouded-dedicated-server/tags
[sknnr-releases]: https://github.com/jsknnr/enshrouded-server/releases
