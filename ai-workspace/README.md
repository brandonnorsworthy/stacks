# ai-workspace

Dedicated sandbox for AI coding agents (Claude Code and opencode) to work on .NET and JavaScript/TypeScript projects, with its own GitHub bot identity. The container idles; you (or an agent runner) attach with `docker exec`. No port is published.

A `db` sidecar (`postgres:16.15`, hostname `db`) lets the AI run integration tests and migrations. It is private to this stack's default network, has no published port and no auth, and its data is disposable. `PGHOST`/`PGUSER`/`PGDATABASE` are preset, so `psql` and Npgsql connection strings like `Host=db;Username=postgres;Database=<name>` work. There is deliberately no Docker socket (it would be root-equivalent), so Testcontainers won't work; use `db` instead.

Source: [`brandonnorsworthy/ai-workspace`](https://github.com/brandonnorsworthy/ai-workspace) (build + push via GitHub Actions in that repo — `stacks` never builds).

## Allocations

- Port: none (access is `docker exec` only)
- UID/GID: `10007:65534` (dedicated stack UID per `AGENTS.md`; `10006` whisper, `10007` = this)
- Tier: `3` cpus / `8gb` for `ai-workspace` (dotnet builds, node tooling); `1` cpu / `4gb` for `db`
- Services: `ai-workspace` and `db` (sandbox Postgres) share UID `10007`
- Image: `ghcr.io/brandonnorsworthy/ai-workspace:sha-c344118` (pinned to the first commit build, which lacks the `psql` client; switch to `0.1.0` once `v0.1.0` is released)

## First-time setup

1. Create a dedicated GitHub bot account (not your personal one), then a **fine-grained PAT** on it limited to the repos the AI should touch (Contents, Pull requests, Issues read/write; Metadata read), with an expiry. Invite the bot as a collaborator on those repos.
2. Create `.env` from `.env.example` and fill in the token and identity, then fix permissions (see root `README.md`):
   ```
   cp .env.example .env
   sudo chown 10001:65534 .env
   setfacl -m u:brandon:rw .env
   chmod 660 .env
   ```
3. Create the data dir owned by the stack UID:
   ```
   mkdir -p data/postgres && sudo chown -R 10007:65534 data
   ```
4. Make sure the GHCR package is readable by the server (public package, or `docker login ghcr.io` on the server).
5. Deploy through Arcane (or `docker compose up -d`).

## Use

```
docker exec -it ai-workspace claude      # first run: log in (or set ANTHROPIC_API_KEY)
docker exec -it ai-workspace opencode    # first run: `opencode auth login`
docker exec -it ai-workspace bash
gh auth status                           # confirms the bot token works
cd /data/workspace && gh repo clone <owner>/<repo>
```

Agent logins, caches, and clones persist in `./data` (`/data/home`, `/data/workspace`).

## Verify

```
docker exec ai-workspace sh -c 'dotnet --version && node --version && claude --version && opencode --version'
docker exec ai-workspace gh auth status
```

## Security notes

- The bot token is the AI's entire blast radius: scope it to specific repos, keep the expiry short, and rotate it by editing `.env` and restarting.
- The sandbox `db` has superuser and `trust` auth, which is safe only because nothing but `ai-workspace` can reach it. Never attach it to the shared `postgres` network or publish its port. The AI has no credentials for the shared server.
- Containers run as its own unprivileged UID, mounts only `./data`, has no Docker socket, no published ports, and no `networks:` (egress to the internet only).
- `GH_TOKEN` is read from the environment at runtime; the entrypoint wires it into git via `gh` and never writes it to disk.

## Deviations

- None.
