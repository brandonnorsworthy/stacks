# stacks

Docker Compose stacks and docs for my homelab containers, managed with [Arcane](https://github.com/getarcaneapp/arcane).

## Purpose

Central place for everything related to running containers on my server:

- `docker-compose.yml` files (one per stack/service)
- Docs: notes on configuration, setup, and any gotchas

## Why

This replaces the old workflow of SSHing into the server and running things by hand (`nano`, `docker compose up -d`, etc). I want to manage my containers declaratively from source control with Arcane as the control plane. No more tribal knowledge living in a shell history.

## Related projects

Supersedes the previous (manual) management repos:

- [norsworthy-monitor](https://github.com/brandonnorsworthy/norsworthy-monitor) — server-side usage recording (CPU/RAM) + start/stop
- [norsworthy-dashboard](https://github.com/brandonnorsworthy/norsworthy-dashboard) — small internal dashboard (side project, not production)

## Stack requirements

Every stack in this repo:

- Runs as its own dedicated UID (not shared with other stacks), with no dedicated group — just the system "nogroup"/"nobody" GID, e.g. `user: "10001:65534"` in `docker-compose.yml`.
- Is scoped to its own stack folder on the host, so a compromised container or user can't escape to other stacks or paths.
- Uses either well-maintained open source images (e.g. `itzg/minecraft`, palworld, valheim, `postgres`, `redis`) or personal images published to Docker Hub / GHCR via a public GitHub Action `docker build`.
- Never bakes production env settings (secrets, API keys, DSNs) into the image or compose file — env comes from the host at runtime. If a stack or image is found with baked-in credentials or other dangerous defaults, say so immediately.

These rules are enforced for AI agents via [AGENTS.md](AGENTS.md).

## Layout

```
stacks/
├── <stack>/
│   ├── docker-compose.yml
│   └── README.md        # per-stack notes (ports, gotchas)
└── docs/                # general docs, runbooks, server info
```
