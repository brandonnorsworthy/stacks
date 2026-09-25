# stacks

Docker Compose stacks and docs for my homelab containers, managed with [Arcane](https://github.com/getarcaneapp/arcane).

## Purpose

Central place for everything related to running containers on my server:

- `compose.yml` files (one per stack/service)
- Docs: notes on configuration, setup, and any gotchas

## Why

This replaces the old workflow of SSHing into the server and running things by hand (`nano`, `docker compose up -d`, etc). I want to manage my containers declaratively from source control with Arcane as the control plane. No more tribal knowledge living in a shell history.

## Related projects

Supersedes the previous (manual) management repos:

- [norsworthy-monitor](https://github.com/brandonnorsworthy/norsworthy-monitor) — server-side usage recording (CPU/RAM) + start/stop
- [norsworthy-dashboard](https://github.com/brandonnorsworthy/norsworthy-dashboard) — small internal dashboard (side project, not production)

## Stack requirements

Every stack in this repo:

- Runs as its own dedicated UID (not shared with other stacks), with no dedicated group — just the system "nogroup"/"nobody" GID, e.g. `user: "10001:65534"` in `compose.yml`.
- Is scoped to its own stack folder on the host, so a compromised container or user can't escape to other stacks or paths.
- Uses either well-maintained open source images (e.g. `itzg/minecraft`, palworld, valheim, `postgres`, `redis`) or personal images published to Docker Hub / GHCR via a public GitHub Action `docker build`.
- Never bakes production env settings (secrets, API keys, DSNs) into the image or compose file — env comes from the host at runtime. If a stack or image is found with baked-in credentials or other dangerous defaults, say so immediately.

These rules are enforced for AI agents via [AGENTS.md](AGENTS.md).

## New stacks

Start every new stack from the central template in [templates/](templates/) (`compose.yml` + `.env.example`) and change the values marked `CHANGE`. Rules:

- Always set `mem_limit` and `cpus`.
- Persistent data goes in `./data`. Named volumes are fine for databases.
- No `networks:` unless the app needs the database or LLM network.
- Pin image versions.
- `.env.example` holds placeholders only; copy to `.env`, which is never committed.

## Environment variables: `.env.example` and `.env`

Every stack has an `.env.example` committed to git. It documents which environment variables the stack needs (names and what they mean) using placeholder values — it is **never** the actual secret values.

The real values live in a `.env` file in the same folder. `.env` is gitignored, so it never touches this repo. It is what the container actually reads (via `env_file: .env` in `compose.yml`) and what gets provisioned on the server (e.g. by Arcane or by setting it manually via the dashboard/monitor tooling).

Workflow when a stack needs new/changed variables:

1. Add or update the variable name in `.env.example` (with a `changeme`-style placeholder).
2. Copy the variable to the server's `.env` for that stack and put in the real value.
3. Restart the stack (Arcane or `docker compose up -d`).

This keeps secrets out of git while still keeping a single source of truth for *which* variables every stack expects.

## Deployment

This repo is cloned onto my home-lab Ubuntu server at `/srv/stacks`. Arcane watches the repo and handles running the stacks — deploys, restarts, scaling — so there's no manual `docker compose up -d` SSH work.

## Layout

```
stacks/
├── templates/           # central compose template for new stacks
├── <stack>/
│   ├── compose.yml
│   ├── .env.example     # placeholders only (committed)
│   ├── .env             # real values (NEVER committed)
│   ├── data/            # persistent data
│   └── README.md        # per-stack notes (ports, gotchas)
└── docs/                # general docs, runbooks, server info
```
