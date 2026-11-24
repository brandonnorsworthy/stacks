# landing

The home page at the network's bare domain: central, device-independent bookmarks grouped on one board, with search, favorites and a status dot per bookmark. Phase 0 of [`brandonnorsworthy/landing`](https://github.com/brandonnorsworthy/landing) (its README has the end goal and phases). The image is built and published by GitHub Actions in that repo; `stacks` never builds.

## Allocations

- Port: `${LANDING_PORT:-10070}` -> container `8080` (default `10070`, the next free slot after spokes-view's `10060`). Host nginx serves the bare domain over HTTPS from it (LAN only).
- UID/GID: `10014:65534` (dedicated stack UID; `10013` = norsworthy-discord-status). Both services run as it. No bind mounts: all data is in Postgres.
- Tier: `0.5` cpus / `2gb` per service.
- Image: `ghcr.io/brandonnorsworthy/landing:<version>`, pinned, the same tag on both services. Rollback = the previous tag (check the migration list first: EF Core migrations only move forward).
- Database: `landing_db` on [`../postgres`](../postgres/README.md) over the external `postgres` network. Roles `landing_migrator` (owner, DDL, used only by `landing-migrate`) and `landing_app` (DML only, used by `landing`).

## Services

| Service | Runs | Gets |
|---|---|---|
| `landing-migrate` | `migrate` once per Up/Redeploy, exits 0 | the migrator's connection string only |
| `landing` | the site, after `landing-migrate` succeeded | the app's connection string only |

Both run read-only with all capabilities dropped and `no-new-privileges`. The site's healthcheck runs `dotnet Landing.Api.dll healthcheck` (the image has no curl), which also checks the database.

## First-time setup

1. `stacks/postgres` must be running.
2. Create the database and roles once, as the Postgres superuser, with `db/bootstrap.sql` from the landing repo. Pass the two passwords as psql variables (`-v migrator_password=... -v app_password=...`); the same values go in `.env`.
3. `cp .env.example .env`, fill in the passwords, then make it readable by Arcane but not by everyone: owner `brandon`, group `65534`, mode `640`.
4. Up from Arcane. `landing-migrate` applies the schema and exits 0, then `landing` turns healthy.

## Security

Phase 0 has no login: anyone who can reach the site can add, edit and delete bookmarks. It is LAN-only: host nginx admits only LAN sources for the site, and port `10070` must not be forwarded to the internet. Login (Phase 1) has to ship before any public exposure.
