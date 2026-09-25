# Agent rules (stacks repo)

Rules for AI agents (and humans) working in this repo.

## Stack requirements

- Each stack MUST run as its own individual, dedicated UID (not shared with other stacks). No dedicated group is needed — use the system "nogroup" GID (e.g. `65534`, the Linux nobody group), e.g. `user: "10001:65534"`.
- The host user owning that stack MUST be scoped to that stack's folder (e.g. via ACLs or `subuid/subgid` + restricted volume mounts) so a compromised container or user cannot escape to other stacks or paths.
- Stacks may reference:
  - well-maintained open source images (e.g. `itzg/minecraft`, palworld, valheim, `postgres`, `redis`), or
  - personal images pushed to Docker Hub or GHCR via a public GitHub Action `docker build`.

## New stacks

Create new stacks from the central template at [`templates/`](templates/README.md) (`compose.yml` + `.env.example`), and follow **all** rules documented in `templates/README.md` (compose conventions, env files, and the managed port allocation scheme). Change the values marked `CHANGE`.

## Security

- No image may be built with baked-in production env settings (secrets, API keys, DSNs) that create security vulnerabilities; env must come from the host at runtime.
- If an existing stack or image is found with baked-in credentials or other dangerous defaults, **mention it immediately** — do not silently leave it alone.
