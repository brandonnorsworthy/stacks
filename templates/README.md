# Docker Compose Template

Copy this folder's `compose.yml` and `.env.example` for every new stack and change the values marked `CHANGE`.

## Rules

- Always set `mem_limit` and `cpus`. Pick the tier that fits the service — these are the only allowed values:
  - `0.5` cpus / `2gb` — very simple services.
  - `1` cpu / `4gb` if it can't get away with less.
  - `3` cpus / `8gb` — websites / APIs.
  - `6` cpus / `16gb` — game servers; bump up only if needed.
  - `12` cpus / `32gb` — reserved for absolutely heavy stuff (e.g. ollama, extremely modded Minecraft).
- Persistent data goes in `./data`. Named volumes are fine for databases.
- No `networks:` unless the app needs the database or LLM network.
- Pin image versions.
- If the image provides health checks, add them to the compose file.
- `.env.example` holds placeholders only; `.env` is never committed.
- Host ports: well-known app ports are fine (e.g. Minecraft `25565`); otherwise claim the next free slot from `10000`, incrementing by `10` per service (`10000`, `10010`, `10020`, …), and record the allocation in the stack's README.
