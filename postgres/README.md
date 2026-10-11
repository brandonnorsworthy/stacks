# postgres

Shared Postgres server for the homelab. Other stacks reach it by hostname `postgres` over the external `postgres` network.

- Image: `postgres:16.15` (pinned to the exact running version, verified 16.15; never change the major version as part of a migration)
- Ports: `127.0.0.1:5432` (well-known port, bound to localhost only; apps connect over the `postgres` network, not the host port)
- UID/GID: `10003:65534` (dedicated stack UID per AGENTS.md; `10001` = arcane, `10002` = valheim)
- Limits: `mem_limit: 4g`, `cpus: 1` (1 cpu / 4gb template tier)
- Data: named volume `postgres`, declared `external: true` so the existing data is reused (and a wrong name fails loudly instead of creating an empty database)
- Network: external `postgres` (must already exist: `docker network create postgres`)
- Env: see `.env.example`; real values live in `.env` on the server only

## First-time setup

1. `cp .env.example .env` and fill in the real `POSTGRES_USER` / `POSTGRES_PASSWORD` (copy the existing values from the old deployment).
2. Confirm the real volume name with `docker volume ls` and set `name:` in `compose.yml`.
3. Pin the image tag to the running version (`docker ps` / `docker inspect postgres`).

## Cutover (from the old compose)

1. `pg_dumpall` backup of everything.
2. Stop the old container with its full grace period: `docker compose stop` (never `down -v`).
3. Chown the data volume to the new UID, with the old container stopped:
    `docker run --rm -v postgres:/d alpine chown -R 10003:65534 /d`
4. Deploy this stack via Arcane.
5. Verify: container healthy (`pg_isready`), dependent apps reconnect, data present.
6. Leave the old volume untouched until verified. Rollback = `chown -R 999:999` the volume (the stock `postgres` user) and start the old compose.

## Migration notes

Changes from the old deployment: added `user` (dedicated UID), added `pg_isready` healthcheck, volume now `external: true`, network renamed `shared-postgres` -> `postgres` (every client app must switch to the new name and `docker network create postgres` first), host port now bound to `127.0.0.1` only. Everything else (container name, restart, grace period, limits) is unchanged.

## Deviations

- None. (Image was initially unpinned; now fixed to `postgres:16.15`, verified against the running server.)
