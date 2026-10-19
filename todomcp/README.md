# todomcp

Homelab's Todo MCP server — a `.NET 10` service that speaks the Model Context Protocol over Streamable HTTP. Lets LibreChat (backed by Ollama) drive todo tools with no human in the loop. Tools: `get_flag`, `add_todo`, `list_todos`, `complete_todo`, `delete_todo`; todos live in the shared Postgres (`todomcp_db`, roles `todomcp_migrator`/`todomcp_app`, joined via the external `postgres` network).

Source: [`brandonnorsworthy/todomcp`](https://github.com/brandonnorsworthy/todomcp) (build + push via GitHub Actions in that repo — `stacks` never builds).

## Allocations

- Port: `10010` (next free slot after arcane's `10000`; container listens on `8080`)
- UID/GID: `10004:65534` (dedicated stack UID per `AGENTS.md`; `10001` arcane, `10002` valheim, `10003` postgres, `10004` = this)
- Tier: `0.5` cpus / `2gb` (very simple service)
- Image: `ghcr.io/brandonnorsworthy/todomcp:0.2.0` (pinned)

## Env

See `.env.example`. `TODOMCP_FLAG` overrides the `get_flag` string; `ConnectionStrings__App` / `ConnectionStrings__Migrator` point at Postgres (create the DB and roles per `docs/policy.md` first).

## Verify (from the server, once pulled)

```
curl -fSs http://localhost:10010/health
curl -fSs -X POST http://localhost:10010/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
curl -fSs -X POST http://localhost:10010/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_flag","arguments":{}}}'
```

Last line must return the configured flag (default `FLAG-lightbulb-1234`).

## Deviations

- None.
