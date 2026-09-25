# Agent rules (stacks repo)

Rules for AI agents (and humans) working in this repo.

## Stack requirements

- Each stack MUST run as its own individual, dedicated UID (not shared with other stacks). No dedicated group is needed — use the system "nogroup" GID (e.g. `65534`, the Linux nobody group), e.g. `user: "10001:65534"`.
- The host user owning that stack MUST be scoped to that stack's folder (e.g. via ACLs or `subuid/subgid` + restricted volume mounts) so a compromised container or user cannot escape to other stacks or paths.
- Stacks may reference:
  - well-maintained open source images (e.g. `itzg/minecraft`, palworld, valheim, `postgres`, `redis`), or
  - personal images pushed to Docker Hub or GHCR via a public GitHub Action `docker build`.

## New stacks

- Create new stacks from the central template at `templates/` (`docker-compose.yml` + `.env.example`); change the values marked `CHANGE`.
- Always set `mem_limit` and `cpus`.
- Persistent data goes in `./data`; named volumes are fine for databases.
- No `networks:` unless the app needs the database or LLM network.
- Pin image versions — no `latest`.
- `.env.example` holds placeholders only; `.env` is never committed (keep it out of git, e.g. via `.gitignore`).

## Security

- No image may be built with baked-in production env settings (secrets, API keys, DSNs) that create security vulnerabilities; env must come from the host at runtime.
- If an existing stack or image is found with baked-in credentials or other dangerous defaults, **mention it immediately** — do not silently leave it alone.
