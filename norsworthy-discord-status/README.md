# norsworthy-discord-status

Posts one Discord message with a status image of the game servers and keeps editing it every 5 minutes: one tile per whitelisted container with its state, uptime, CPU, memory and a 6-hour usage graph; stopped servers get a dimmed "Offline" tile. People outside the network can see server status in Discord without any API being exposed. It reads the stats from `norsworthy-monitor`.

Source: [`brandonnorsworthy/dn-norsworthy-discord-status`](https://github.com/brandonnorsworthy/dn-norsworthy-discord-status) (the image is built and pushed by GitHub Actions in that repo; `stacks` never builds).

## Allocations

- Port: `3500` -> container `8080`. Kept from the old deployment instead of a `100x0` slot (see Migration notes). Endpoints: `/status.png` (the grid), `/status.png?container=<name>` (one card, whitelisted containers only), `/health`.
- UID/GID: `10013:65534` (dedicated stack UID; `10012` = spokes-view). This overrides the image's own `app` user (1654), so `./data` must be owned by `10013`.
- Tier: `0.5` cpus / `2gb` (very simple service).
- Image: `ghcr.io/brandonnorsworthy/dn-norsworthy-discord-status:1.2.0` (pinned; bump deliberately, rollback = the previous tag).
- Data: `./data` bind mount: `message-id.txt`, the id of the Discord message being edited. Delete it and the service posts a new message.
- Network: external `norsworthy-net` (must exist before `Up`; `norsworthy-monitor` is on it).

## Start / stop

First time, create the data folder owned by the stack UID (Docker would otherwise create it as root and the message id could not be saved, so every restart would post a new message):

```
sudo mkdir -p data && sudo chown 10013:65534 data
cp .env.example .env    # then fill it in; Arcane must be able to read it
```

Then start and stop the stack from Arcane (Up / Down). The healthcheck runs the app's own `healthcheck` command inside the container (the image has no curl); it turns `healthy` in under a minute.

## Env

See `.env.example`. `DISCORD_WEBHOOK_URL` is a secret and lives only in the server's `.env`. `BASE_URL` is the monitor API, `CONTAINER_WHITELIST` the containers that get a tile (a name that matches no container gets none).

**When you add a game server, add its container name to `CONTAINER_WHITELIST` in the server's `.env` and Redeploy this stack**, or it will not appear in the Discord message. Retired servers can stay listed: they show as dimmed Offline tiles.

## Verify (from the server)

```
curl -fsS http://localhost:3500/health                      # Healthy
curl -fsS -o /tmp/status.png http://localhost:3500/status.png && file /tmp/status.png
docker logs norsworthy-discord-status 2>&1 | tail          # "Message edited. ID: ..."
```

Then look at the Discord message: it should show the grid and a fresh "Generated at" time.

## Security

- No authentication on port `3500`. It only serves the status image of whitelisted containers; keep it on the LAN.
- The webhook URL is a credential: anyone holding it can post to the channel. The app does not log it (image 1.0.0 and later).

## Migration notes

Moved from `/srv/websites/dn-norsworthy-discord-status`, where the image was built on the server (`build: .`), into this stack on 2026-11-15.

- `build:` replaced by the pinned GHCR image.
- Container port changed from `3500` to `8080` (the .NET default); the host port stays `3500`.
- Limits changed from `512m / 1 cpu` to the `0.5 / 2g` tier. `stop_grace_period` from `90s` to `30s` (nothing to flush).
- Now runs as `10013:65534` instead of root, with a healthcheck.
- `data/message-id.txt` is copied from the old folder so the existing Discord message keeps being edited.
- The old folder is left in place as the rollback: stop this stack in Arcane, then `docker compose up -d` there.

## Deviations

- `networks:` uses `norsworthy-net`, which is neither the database nor the LLM network the template allows. It is the existing external network `norsworthy-monitor` is on, and the service needs it to reach the monitor API by name. Goes away if the monitor is ever reached another way.
- Host port `3500` instead of the next `100x0` slot, kept per `MIGRATION.md` (existing ports stay).

## Testing note

Tested on the dev PC (2026-11-15) with the published `1.0.0` image, started from the Arcane web UI (`Up`), with a local `norsworthy-net` network, `BASE_URL` pointed at the server's monitor over the LAN, and a test webhook: container `healthy` as `10013:65534` on `norsworthy-net`; `/health` 200; `/status.png` a 1280x1260 grid of six whitelisted servers (four online, two offline); `?container=arcane` (not whitelisted) 404; log `Message edited. ID: ...` and the test Discord message updated in place; no webhook URL in the logs. Stopped again with `Down`.

Re-tested on the dev PC (2026-11-16) with the published `1.1.0` image (plain "up for N weeks" uptime and a green/red health heart), same method: `Up` from the dev Arcane UI, container `healthy` as `10013:65534`, `/health` 200, `/status.png` a 1280x1260 grid of six servers with hearts and "up for" text, `?container=arcane` 404, log `Message edited. ID: ...`. Stopped again with `Down`.

Re-tested on the dev PC (2026-11-24) with the published `1.2.0` image (offline tiles collapsed to half height, two per tile slot), started from the dev Arcane UI (`Up`), with the full game-server whitelist against the live monitor: container `healthy` as `10013:65534`, `/health` 200, `/status.png` a 1920x1260 grid (4 online, 8 offline), `?container=atm11` 200, `?container=arcane` 404, log `Message edited. ID: ...`.
