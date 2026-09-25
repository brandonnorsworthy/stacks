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

## Layout

```
stacks/
├── <stack>/
│   ├── docker-compose.yml
│   └── README.md        # per-stack notes (ports, gotchas)
└── docs/                # general docs, runbooks, server info
```
