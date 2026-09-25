# Docker Compose Template

Copy this folder's `compose.yml` and `.env.example` for every new stack and change the values marked `CHANGE`.

## Rules

- Always set `mem_limit` and `cpus`.
- Persistent data goes in `./data`. Named volumes are fine for databases.
- No `networks:` unless the app needs the database or LLM network.
- Pin image versions.
