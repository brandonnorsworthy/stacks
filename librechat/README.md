# librechat

LibreChat web UI (API + client) backed by the self-hosted Ollama server, with MongoDB and Meilisearch (chat search). Registers the `todomcp` stack as an MCP server.

## Allocations

- Port: `10020` -> container `3080` (next free slot after todomcp's `10010`)
- UID/GID: `10005:65534` (dedicated stack UID; `10001` arcane, `10002` valheim, `10003` postgres, `10004` todomcp, `10005` = this; all three services share it)
- Images (pinned): `ghcr.io/danny-avila/librechat:v0.8.8`, `mongo:8.0.20`, `getmeili/meilisearch:v1.35.1`
- Limits: librechat `3 cpus / 8g`, mongodb `1 / 4g`, meilisearch `0.5 / 2g` (template tiers)
- Data: external named volumes `librechat-mongo`, `librechat-meili`, `librechat-images`, `librechat-uploads`, `librechat-logs` (must exist and be owned by `10005:65534`)
- Config: `librechat.yaml` (committed, no secrets); secrets in `.env` (server only, see `.env.example`)

## First-time setup

1. `cp .env.example .env`, fill in real secrets (`openssl rand -hex 32` for keys, `-hex 16` for `CREDS_IV`) and set `DOMAIN_CLIENT`/`DOMAIN_SERVER`.
2. Create the volumes and chown them:
    `for v in mongo meili images uploads logs; do docker volume create librechat-$v; done`
    `docker run --rm $(for v in mongo meili images uploads logs; do echo -v librechat-$v:/d/$v; done) alpine chown -R 10005:65534 /d`
3. Edit `librechat.yaml`: set the default Ollama model, and confirm Ollama's address (see Ollama below).
4. Deploy via Arcane, open `http://<host>:10020`, register your account, then set `ALLOW_REGISTRATION=false` and redeploy.

## Ollama

`librechat.yaml` points at `http://host.docker.internal:11434`, which resolves to the host via `extra_hosts`. This works if Ollama is a published port on the server. Ollama must listen on an address reachable from containers (not just `127.0.0.1`). If Ollama runs in a container on a shared LLM network, join that network (external) in `compose.yml` and change the URL to its container name (update `allowedAddresses` too).

For MCP tools, use an Ollama model that supports tool calling.

## Speech to text (mic button)

Firefox has no built-in speech recognition, so `librechat.yaml` (`speech:`) records audio and sends it to the `whisper` stack at `host.docker.internal:10030`. Deploy that stack first (see `../whisper/README.md`).

## Cutover (from the old LibreChat)

Not done from here, since no SSH. Old data is MongoDB (conversations/users) plus uploads.

1. `mongodump` the old database (or copy the old Mongo data dir while the old container is stopped).
2. Stop the old compose (`docker compose stop`, never `down -v`); ports and container names must be free.
3. Copy the old Mongo data dir into the `librechat-mongo` volume, uploads into `librechat-uploads`, images into `librechat-images`, then chown the volumes to `10005:65534`. Copy the old `CREDS_KEY`/`CREDS_IV`/`JWT_*` into `.env` (changing them invalidates stored credentials and sessions).
4. Deploy, verify healthy, log in, check history. Meilisearch reindexes on its own.
5. Keep the old data untouched until verified; rollback = start the old compose.

## Migration notes

Written from upstream's official compose + template, **not** from the old running compose (not accessible). Compare against it before cutover: ports, env vars, the old `librechat.yaml`, and the old image tag. Check `docker ps` for the running version and pin to it if you want to avoid a version jump.

## Deviations

- Dropped upstream's `rag_api` + `vectordb` (pgvector) and admin panel; add later if you use file RAG.
- MongoDB runs without auth (`--noauth`, as upstream does), bound to this stack's private network only and not published.
- Ollama and todomcp are reached via the host (`host.docker.internal`) rather than a shared Docker network.
