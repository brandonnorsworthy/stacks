# spokes

[Spokes](https://github.com/brandonnorsworthy/spokes): one self-hosted tree of to-dos (areas, projects, steps) that any MCP-capable model can read and drive over Streamable HTTP at `/mcp`. A single Rust binary with its SQLite database in `./data`. Tools: `add`, `get`, `next`, `set_status`, `update`, `move`, `search`, `focus`, `claim`, `release`, `ask`, `answer`.

Source: [`brandonnorsworthy/spokes`](https://github.com/brandonnorsworthy/spokes) (the image is built and pushed by GitHub Actions in that repo; `stacks` never builds).

## Allocations

- Port: `${SPOKES_PORT:-10050}` -> container `8787` (default `10050`, the next free slot after dbhub's `10040`), endpoints `/mcp`, `/api` (read-only JSON, from 0.2.0; `/api/history` for replay from 0.4.0, `/api/search` for the viewer's search box from 0.5.0) and `/health`.
- UID/GID: `10010:65534` (dedicated stack UID; `10009` = python-sandbox). This overrides the image's own `nonroot` user (65532), so `./data` must be owned by `10010`.
- Tier: `0.5` cpus / `2gb` (very simple service).
- Image: `ghcr.io/brandonnorsworthy/spokes:0.5.0` (pinned; bump deliberately, rollback = the previous tag; 0.5.0 changed no stored data, so `0.4.0` is a clean rollback).
- Data: `./data` bind mount: `spokes.db` (+ `-wal`/`-shm`) and `backups/`.

## Start / stop

First time, create the data folder owned by the stack UID. Docker would otherwise create it as root, and spokes exits with `unable to open database file`:

```
sudo mkdir -p data && sudo chown 10010:65534 data
cp .env.example .env    # then set ownership/ACL as in ../README.md (Arcane must read it)
```

Then start and stop the stack from Arcane (Up / Down). The healthcheck runs `spokes healthcheck` inside the container (the distroless image has no shell or curl); it turns `healthy` within a few seconds.

## Env

See `.env.example`. No secrets: `SPOKES_TZ` (what "today" is for due dates, deferrals and repeats), `SPOKES_BACKUP_DIR` / `SPOKES_BACKUP_KEEP` (nightly backups, 14 kept), and optional `SPOKES_ALLOWED_HOSTS`, `SPOKES_ALLOWED_ORIGINS`, `SPOKES_LOG`. `SPOKES_BIND` and `SPOKES_DB_PATH` are set by the image.

## Security

`/mcp` and the read-only JSON API at `/api` (used by [spokes-view](../spokes-view/README.md), from 0.2.0) have no authentication; anyone who can reach port `10050` can read and change the to-dos. Requests with an `Origin` header are refused unless listed in `SPOKES_ALLOWED_ORIGINS` (blocks browser-driven attacks); MCP clients send none. Keep the port on the LAN (not forwarded through nginx to the internet) until auth exists.

## HTTPS name (LAN only)

`https://spokes.zcog.net` reaches the same endpoints through host nginx's generic `*.zcog.net` vhost (wildcard cert, LAN sources only, buffering off for streaming). The DNS record lives in the `dns` repo and the nginx files in `docs/zcog.net/nginx/`. If you change `SPOKES_PORT`, regenerate the nginx map line. The plain `http://<server>:10050` address still works.

## Clients

- Claude Code: `claude mcp add --transport http spokes http://<server>:10050/mcp` (or a `"type": "http"` entry in `~/.claude.json`).
- opencode (`~/.config/opencode/opencode.jsonc`): `"spokes": {"type": "remote", "url": "http://<server>:10050/mcp"}`.
- LibreChat (`../librechat/librechat.yaml` -> `mcpServers`): `spokes: {type: streamable-http, url: http://host.docker.internal:10050/mcp, timeout: 30000}`, then enable it with the `MCP Servers` button. If `SPOKES_ALLOWED_HOSTS` is set, include `host.docker.internal`.

## Verify (from the server)

```
curl -fsS http://localhost:10050/health        # ok
curl -fsS -X POST http://localhost:10050/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-11-25' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"next","arguments":{}}}'
```

`initialize` reports `serverInfo.version` (the image version).

## Backup and restore

Backups are written inside the container by spokes itself (SQLite `VACUUM INTO`, consistent while running): one at startup, then every 24h, to `/data/backups` (`./data/backups` on the host), keeping the newest `SPOKES_BACKUP_KEEP`.

- Extra backup now: `docker exec spokes spokes backup /data/backups`
- Restore (stop spokes first: `compose run` starts a separate container, which cannot see a running server, so it will not stop you):
  1. Stop the stack in Arcane.
  2. `docker compose run --rm --no-deps spokes restore /data/backups/<file>.db`
  3. Start the stack in Arcane.

  The current database is moved aside to `/data/spokes.db.before-restore-<time>` first, so a restore can be undone.

## Testing note

Tested on the dev PC (2026-11-14) with a scratch copy of this folder and the locally built image (`spokes:0.1.0-local`, since `0.1.0` is not published yet), `./data` chowned to `10010:65534`, and `:Z` added to the bind mount (Fedora SELinux only; the Ubuntu server needs no label): container `healthy` as `10010:65534`; `initialize`, `tools/list` (12 tools), `add` and `next` over `http://127.0.0.1:10050/mcp`; database and backups in `./data`, rotation, `spokes backup` via `docker exec`, `restore` via `compose run`, data intact after restart. With `./data` not owned by `10010` the container fails with `unable to open database file`. Still to do per `docs/server-standards.md` rule 6: publish `v0.1.0`, then start it from the Arcane web UI and repeat the calls.

## Deviations

- Rust and SQLite instead of the default .NET/Postgres stack (recorded in `docs/projects.md`); no `networks:` since it needs no database or LLM network.
