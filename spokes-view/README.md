# spokes-view

[spokes-view](https://github.com/brandonnorsworthy/spokes-view): the visual companion for [Spokes](../spokes/README.md). Pick a Spokes server and browse its to-do tree as a radial graph in a browser, with no LLM involved. Read-only. One Rust binary with the page built in; it keeps no data.

Source: [`brandonnorsworthy/spokes-view`](https://github.com/brandonnorsworthy/spokes-view) (the image is built and pushed by GitHub Actions in that repo; `stacks` never builds).

## Allocations

- Port: `${SPOKES_VIEW_PORT:-10060}` -> container `8788` (default `10060`, the next free slot after spokes' `10050`). The page is `/`, the probe is `/health`.
- HTTPS name (LAN only): `https://view.zcog.net`, through host nginx's generic `*.zcog.net` vhost (wildcard cert, LAN sources only). The DNS record is in the `dns` repo and the nginx files are in `docs/zcog.net/nginx/`. If you change `SPOKES_VIEW_PORT`, regenerate the nginx map line.
- UID/GID: `10012:65534` (dedicated stack UID; `10011` = zomboid). Overrides the image's own `nonroot` user (65532).
- Tier: `0.5` cpus / `2gb` (very simple service).
- Image: `ghcr.io/brandonnorsworthy/spokes-view:0.3.0` (pinned; bump deliberately, rollback = the previous tag). 0.2.0 adds Replay, 0.3.0 adds search (find a node and jump to it).
- Data: none. No volume; the container filesystem is read-only.

## Needs

Every server listed in `SPOKES_VIEW_SERVERS` must run Spokes `0.2.0` or later (the read-only `/api`). An older server shows "older than 0.2.0 and has no read API" in the page. Replay needs Spokes `0.4.0` or later and search needs `0.5.0` or later; against an older server the button or search box is disabled with that note and everything else works. If a Spokes server sets `SPOKES_ALLOWED_HOSTS`, it must include the host name or IP used in `SPOKES_VIEW_SERVERS`.

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
