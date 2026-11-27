# spokes-view

[spokes-view](https://github.com/brandonnorsworthy/spokes-view): the visual companion for [Spokes](../spokes/README.md). Pick a Spokes server and browse its to-do tree as a radial graph in a browser, with no LLM involved. Read-only. One Rust binary with the page built in; it keeps no data.

Source: [`brandonnorsworthy/spokes-view`](https://github.com/brandonnorsworthy/spokes-view) (the image is built and pushed by GitHub Actions in that repo; `stacks` never builds).

## Allocations

- Port: `${SPOKES_VIEW_PORT:-10060}` -> container `8788` (default `10060`, the next free slot after spokes' `10050`). The page is `/`, the probe is `/health`.
- HTTPS name (LAN only): `https://view.<domain>`, through host nginx's generic `*.<domain>` vhost (wildcard cert, LAN sources only). The DNS record is in the `dns` repo and the nginx files are in `docs/<domain>/nginx/`. If you change `SPOKES_VIEW_PORT`, regenerate the nginx map line.
- UID/GID: `10012:65534` (dedicated stack UID; `10011` = zomboid). Overrides the image's own `nonroot` user (65532).
- Tier: `0.5` cpus / `2gb` (very simple service).
- Image: `ghcr.io/brandonnorsworthy/spokes-view:0.14.2` (pinned; bump deliberately, rollback = the previous tag, 0.9.0). 0.14.2 (2026-11-27, mobile-first epic SPOKE-1017; 0.10 to 0.13 were merged but never deployed on their own) is the mobile-first redesign: a bottom tab bar (Tree, Board, Backlog, Search, Tracked), an Actions sheet on every node instead of right-click (Isolate, View board/backlog, Expand all below, Copy ID/link), boards one column at a time on a phone with collapsible, capped columns on desktop, a details bottom sheet, touch gestures on the tree, filters kept in the URL, and a Tracked tab listing nodes tracked through MCP with their progress (needs Spokes 0.15.0; deploy Spokes first). 0.9.0 adds search by ID (an ID badge; typing `958` jumps to SPOKE-958), matched-keyword chips, a "More results" row and ticks Show done / Show dropped when a picked hit needs it (needs Spokes 0.14.0 for all of it; deploy Spokes first). 0.8.1 shows and filters no refinement on closed tasks. 0.8.0 adds "Expand all below", a stronger epic style and a one-row tree header. 0.7.2 leaves areas off the Kanban board. 0.7.1 adds Kanban and Backlog views, click to focus, and type and refinement badges (the badges need Spokes 0.10.1). 0.5.0 adds a Source section (the user's raw words, from Spokes 0.7.0) to the detail panel; it shows nothing against an older Spokes. 0.2.0 adds Replay, 0.3.0 adds search (find a node and jump to it), 0.4.0 replaces "Show closed" with separate "Show done" and "Show dropped" boxes, both off by default (0.4.1 only adds the accepted base-image CVE exception).
- Data: none. No volume; the container filesystem is read-only.

## Needs

Every server listed in `SPOKES_VIEW_SERVERS` must run Spokes `0.2.0` or later (the read-only `/api`). An older server shows "older than 0.2.0 and has no read API" in the page. Replay needs Spokes `0.4.0` or later, search needs `0.5.0` or later and the done/dropped boxes need `0.6.0` or later (deploy Spokes first: an older one answers 400 and the page shows "the Spokes server answered 400"); against an older server the button or search box is disabled with that note and everything else works. If a Spokes server sets `SPOKES_ALLOWED_HOSTS`, it must include the host name or IP used in `SPOKES_VIEW_SERVERS`.

## Start / stop

```
cp .env.example .env    # then set ownership/ACL as in ../README.md (Arcane must read it)
```

Then start and stop the stack from Arcane (Up / Down). It turns `healthy` within a few seconds.

## Env

See `.env.example`. No secrets: `SPOKES_VIEW_SERVERS` (the picker's `Name=URL` list) and optional `SPOKES_VIEW_LOG`. `SPOKES_VIEW_BIND` is set by the image. A malformed server list stops the container at startup with a log line naming the bad entry.

## Security

No authentication: anyone who can reach port `10060` can read every to-do and note on the listed servers. It cannot change anything (it only calls Spokes' read-only API), and the browser can only pick a configured server by name, never supply a URL. Keep the port on the LAN (not forwarded through nginx to the internet) until auth exists.

## Verify

Open `http://<server>:10060` in a browser: the picker lists the configured servers and the graph of the first one draws. From a shell:

```
curl -fsS http://localhost:10060/health          # ok
curl -fsS http://localhost:10060/api/servers     # each server with status "ok" and its version
```
