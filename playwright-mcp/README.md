# playwright-mcp

[Playwright MCP](https://github.com/microsoft/playwright-mcp) browser server (headless Chromium) for LLM clients: LibreChat, Claude Code, opencode. Same setup the server's LibreChat already uses.

## Allocations

- Port: `127.0.0.1:8931` -> container `8931` (Playwright's well-known default port, so no claim from the `10000+` pool), endpoint `/mcp`. Loopback only. LibreChat reaches it as `http://playwright-mcp:8931/mcp` over the `mcp` network.
- UID/GID: `10008:65534`
- Tier: `0.5` cpus / `2g`
- Image: `mcr.microsoft.com/playwright/mcp:v0.0.83` (pinned)
- Network: external `mcp` (create once: `docker network create mcp`). Only LibreChat and this stack join it.

## Safety

No auth on the endpoint, `--no-sandbox` (the container is the boundary), and `--allowed-hosts "*"` (needed so the container name works as a Host header). The browser can reach anything this container's network can, including LAN addresses. Keep it off published non-loopback ports. `--isolated` keeps each session's profile in memory, so nothing persists.

## Clients

- Claude Code (`~/.claude.json`, server `browser`): `http://127.0.0.1:8931/mcp`.
- LibreChat: `librechat.yaml` -> `browser` -> `http://playwright-mcp:8931/mcp`; enable with the `MCP Servers` button in the chat input.

## Env

- `PLAYWRIGHT_MCP_PING_TIMEOUT_MS=0` (set in `compose.yml`): turns off the server's heartbeat. Playwright MCP pings each HTTP client every 3 s and closes the session if no reply arrives within 5 s (default). LibreChat's MCP client never answers those pings, so without this every tool call ended with `404 with active session - session lost` / `MCP error -32000: Connection closed`, whatever the URL. Claude Code's client does answer, which is why it never showed the problem. Found and verified 2026-10-29 (LibreChat v0.8.7, Playwright MCP v0.0.83).
