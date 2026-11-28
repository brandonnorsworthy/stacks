# spokes

[Spokes](https://github.com/brandonnorsworthy/spokes): one self-hosted tree of to-dos (areas, projects, steps) that any MCP-capable model can read and drive over Streamable HTTP at `/mcp`. A single Rust binary with its SQLite database in `./data`. Tools: `add`, `get`, `next`, `set_status`, `update`, `move`, `search`, `focus`, `claim`, `release`, `ask`, `answer`.

Source: [`brandonnorsworthy/spokes`](https://github.com/brandonnorsworthy/spokes) (the image is built and pushed by GitHub Actions in that repo; `stacks` never builds).

## Allocations

- Port: `${SPOKES_PORT:-10050}` -> container `8787` (default `10050`, the next free slot after dbhub's `10040`), endpoints `/mcp`, `/api` (read-only JSON, from 0.2.0; `/api/history` for replay from 0.4.0, `/api/search` for the viewer's search box from 0.5.0, `include_dropped` for its "Show dropped" box from 0.6.0) and `/health`.
- UID/GID: `10010:65534` (dedicated stack UID; `10009` = python-sandbox). This overrides the image's own `nonroot` user (65532), so `./data` must be owned by `10010`.
- Tier: `0.5` cpus / `2gb` (very simple service).
- Image: `ghcr.io/brandonnorsworthy/spokes:0.29.1` (pinned; bump deliberately). 0.29.1 (2026-11-27, SPOKE-260): a node may not share its title (case, accents, punctuation and spacing ignored) with an unfinished sibling: `add`, `move` and an `update` title that would make a duplicate are refused, naming the existing node (done and dropped siblings never block); `add` lists up to three likely duplicates anywhere in the tree, and an `add` without an agent label gets a hint asking for one. No schema change: rollback = pin 0.28.2. 0.28.2 (2026-11-27, SPOKE-1110): `SPOKES_HINTS=0` makes no hint lines instead of cutting every line starting `Next: ` out of replies, so a note or source line the user wrote that starts that way comes back. Production runs with hints on, so replies there are unchanged. No schema change: rollback = pin 0.28.1. 0.28.1 (2026-11-27, SPOKE-1126): the close refusal names which tasks are claimed or waiting, writes every listed id out, and with 100 or more suggests searches that work. No schema change: rollback = pin 0.28.0. 0.28.0 (2026-11-27, SPOKE-345): `set_status` refuses `done` and `dropped` while anything under the node is unfinished, and `open` under closed work; `search` takes `state: under_closed` to list what older data still holds (after upgrading an older database, move, close or reopen above whatever it lists). No schema change: rollback = pin 0.27.0. 0.27.0 (2026-11-27, SPOKE-1029): `search` lists leftover work with `state` (`stalled`, `claimed`, `in_progress`, `waiting`, `open`, `children_done`; `status` accepted), `claimed_by` and `untouched_days`, with an empty query listing every match and each line naming the holder and claim expiry. No schema change: rollback = pin 0.26.0. 0.26.0 (2026-11-27, SPOKE-551): `claim` is refused only when another agent holds that same task, so an orchestrator can claim a project after a worker has started a step; the reply names what others hold beneath it. No schema change: rollback = pin 0.25.1. 0.25.1 (2026-11-27, SPOKE-1079) adds the write-reply hints: `add`, `claim`, `release`, `ask`, `answer`, `move` and `set_status dropped` end with a `Next: ` line when the node state calls for one (0.19.0 to 0.25.1 were released together as 0.25.1). No schema change: rollback = pin 0.18.0. 0.18.0 (2026-11-27, SPOKE-692): `claim` on a task with open children ends with a `Next: ` line naming them (the next step of an ordered parent, or up to three of an unordered one) and saying to claim the subtask too. No schema change: rollback = pin 0.17.0. 0.17.0 (2026-11-27, SPOKE-1006) adds claim hints: `get` on an open task nobody holds, `next` when it lists work, and `update notes_append` on an unheld task end with a `Next: ` line telling the agent to claim before working, and the set_status next-step hint adds "Claim it before starting.". No schema change: rollback = pin 0.16.0. 0.16.0 (2026-11-27, SPOKE-1078) moves the next-action hints into one module: hint lines in replies now start with `Next: `, `guide` explains them, and `SPOKES_HINTS=0` turns them off. No schema change: rollback = pin 0.15.0. 0.15.0 (2026-11-27, mobile-first epic SPOKE-1017) adds a `tracked` mark on nodes (schema version 6, a nullable `tracked_at` column), set only through MCP (`update` or `add` with `tracked: true`, `false` clears it; `search` takes `tracked: true` as a filter) and listed read-only at `/api/tracked` for spokes-view's Tracked tab. **Schema change:** rollback = stop, `spokes restore` the pre-upgrade backup (`data/backups/spokes-20261127-183508.db`), pin 0.14.1 (writes since are lost). 0.11.0 to 0.14.1 (search epic SPOKE-980) find a node by its id (`958`, `SPOKE-958`), rank loose queries by coverage of the rarer keywords with `match` best/any/all, show short paths with ancestor ids and page with `offset`, and teach agents this in the connect instructions and `guide`; `/api/search` gains `id_match`, `matched`, `hidden_by`, `match`, `weaker_hidden` and `offset` (additive). No schema change: rollback = pin 0.10.2. 0.10.2 shows no refinement on done and dropped tasks (no schema change; rollback = pin 0.10.1). 0.10.1 adds optional `type` and `refinement` fields and the `guide` tool, and moves the schema to version 5, so going back to 0.7.0 also means `spokes restore` of a backup taken before the upgrade. **Rollback is no longer just the previous tag:** 0.7.0 moved the database schema to version 4 (the `source` column), and an older server refuses a newer schema. To go back to 0.6.x, stop the stack, `spokes restore` a backup taken before the upgrade, then pin the older tag (writes made since are lost). Before 0.7.0, 0.5.0 and 0.6.x changed no stored data, so each was a clean rollback. 0.7.0 adds the `source` field (the user's raw words on a node) and the `migrate-source` command (`docker exec spokes /usr/local/bin/spokes migrate-source`, dry run unless `--apply`). `0.6.0` was tagged but its release scan failed (base-image CVE) and it was never deployed; `0.6.1` carries the accepted, expiring exception (see `docs/projects.md`).
- Data: `./data` bind mount: `spokes.db` (+ `-wal`/`-shm`) and `backups/`.

## Start / stop

First time, create the data folder owned by the stack UID. Docker would otherwise create it as root, and spokes exits with `unable to open database file`:

```
sudo mkdir -p data && sudo chown 10010:65534 data
cp .env.example .env    # then set ownership/ACL as in ../README.md (Arcane must read it)
```

Then start and stop the stack from Arcane (Up / Down). The healthcheck runs `spokes healthcheck` inside the container (the distroless image has no shell or curl); it turns `healthy` within a few seconds.

## Env

See `.env.example`. No secrets: `SPOKES_TZ` (what "today" is for due dates, deferrals and repeats), `SPOKES_BACKUP_DIR` / `SPOKES_BACKUP_KEEP` (nightly backups, 14 kept), and optional `SPOKES_ALLOWED_HOSTS`, `SPOKES_ALLOWED_ORIGINS`, `SPOKES_HINTS` (`0` turns hint lines off), `SPOKES_LOG`. `SPOKES_BIND` and `SPOKES_DB_PATH` are set by the image.

## Security

`/mcp` and the read-only JSON API at `/api` (used by [spokes-view](../spokes-view/README.md), from 0.2.0) have no authentication; anyone who can reach port `10050` can read and change the to-dos. Requests with an `Origin` header are refused unless listed in `SPOKES_ALLOWED_ORIGINS` (blocks browser-driven attacks); MCP clients send none. Keep the port on the LAN (not forwarded through nginx to the internet) until auth exists.

## HTTPS name (LAN only)

`https://spokes.<domain>` reaches the same endpoints through host nginx's generic `*.<domain>` vhost (wildcard cert, LAN sources only, buffering off for streaming). The DNS record lives in the `dns` repo and the nginx files in `docs/<domain>/nginx/`. If you change `SPOKES_PORT`, regenerate the nginx map line. The plain `http://<server>:10050` address still works.

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
