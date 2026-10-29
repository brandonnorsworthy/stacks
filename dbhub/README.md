# dbhub

[DBHub](https://github.com/bytebase/dbhub) MCP server giving LLM clients (Claude Code, opencode, LibreChat) **read-only** SQL access to the shared dev Postgres (`digitalhearthdb`, `todomcp_db`).

## Allocations

- Port: `127.0.0.1:10040` -> container `8080`, endpoint `/mcp` (loopback only; DBHub has no auth). LibreChat reaches it as `http://dbhub:8080/mcp` over the `postgres` network.
- UID/GID: `10006:65534`
- Tier: `0.5` cpus / `512mb`
- Image: `bytebase/dbhub:1.5.0` (pinned)

## Safety

Two layers: the TOML sets `readonly = true` + `max_rows` on each `execute_sql` tool, and the Postgres role `mcp_readonly` only has `CONNECT`/`USAGE`/`SELECT` (plus `default_transaction_read_only`). The role is the real boundary.

## New database / new tables

Per database: `GRANT CONNECT ON DATABASE x TO mcp_readonly; GRANT USAGE ON SCHEMA public TO mcp_readonly; GRANT SELECT ON ALL TABLES IN SCHEMA public TO mcp_readonly;` and `ALTER DEFAULT PRIVILEGES FOR ROLE <owner> ...` for future tables; then add a `[[sources]]` + `[[tools]]` pair.

## Env

See `.env.example`.

## Arcane

`.env` must be readable by Arcane's UID (10001): keep it `chmod 660` (the ACL entry exists), not `600`, or Arcane shows "Environment file unreadable" and disables Up/Down.

## Clients

- Claude Code (`~/.claude.json`) and opencode (`~/.config/opencode/opencode.jsonc`): `http://127.0.0.1:10040/mcp`. For opencode, `opencode mcp list` reports nothing; use `opencode run --standalone`.
- LibreChat: `librechat.yaml` -> `http://dbhub:8080/mcp`; enable with the `MCP Servers` button in the chat input.
