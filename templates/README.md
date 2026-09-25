# Docker Compose Template

Copy this folder's `compose.yml` and `.env.example` for every new stack and change the values marked `CHANGE`.

## Rules

- Always set `mem_limit` and `cpus`.
- Persistent data goes in `./data`. Named volumes are fine for databases.
- No `networks:` unless the app needs the database or LLM network.
- Pin image versions.
- `.env.example` holds placeholders only; `.env` is never committed.
- Host ports: well-known app ports are fine (e.g. Minecraft `25565`); otherwise claim the next free slot from `10000`, incrementing by `10` per service (`10000`, `10010`, `10020`, …), and record the allocation in the stack's README.
