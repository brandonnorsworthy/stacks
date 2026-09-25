# valheim

Valheim dedicated server. Image `lloesche/valheim-server` (Docker Hub), pinned.

- Image: `lloesche/valheim-server:sha-e36cbfb0ddc8` (pinned)
- Ports: UDP `2456` (game), `2457` (query), `2458` (crossplay/RPC). These are Valheim's **well-known** Steam ports, so no claim from the `10000+` managed pool is needed — recorded here per the allocation rule. 2458 is only actively used when `CROSSPLAY=true` or a mod uses RPC (gameport+2).
- UID/GID: `10002:65534` (dedicated stack UID per AGENTS.md, first app-stack slot after `10001`=arcane; `nogroup` GID). The image entrypoint starts as root to `chown` `/config` and `/opt/valheim`, then drops to `PUID:PGID` — so `PUID=10002`/`PGID=65534` in `.env` is what enforces the dedicated UID (compose `user:` would skip the entrypoint and break first-run setup).
- Limits: `mem_limit: 16g`, `cpus: 6.0` (game-server tier per the template rules; upstream: ~2.8 GB RSS idle, min 2 core/4 GB, recommended 4 core/8 GB).
- Data: `./data/config` → `/config` (worlds, mods, backups) and `./data/opt-valheim` → `/opt/valheim` (cached server binaries). Both under `./data` (gitignored, persist), per the data-dir rule.

## First-time setup

1. Create `.env` and set real values:

   ```sh
   cp .env.example .env
   # then edit SERVER_NAME, WORLD_NAME, SERVER_PASS, TZ, CROSSPLAY, etc.
   ```

   `SERVER_PASS` must be ≥ 5 chars. Leave `SERVER_PASS_FILE` commented unless feeding a Docker secret.

2. Start:

   ```sh
   docker compose up -d
   ```

   First boot downloads the server + steamcmd into `./data/opt-valheim`; subsequent starts reuse it.

3. (Optional) pre-create the world dir and own it as the stack UID (the entrypoint does this too):

   ```sh
   mkdir -p data/config/worlds_local
   sudo chown -R 10002:65534 data
   ```

## Notes

- **Vanilla vs mods:** `VALHEIM_PLUS` and `BEPINEX` are mutually exclusive — set at most one to `true`. Mod/plugins/patchers go under `/config/valheimplus/` or `/config/bepinex/` respectively (drop files into `data/config/...`).
- **Backups:** written to `/config/backups` (i.e. `data/config/backups`, persisted on the host). Do **not** point `BACKUPS_DIRECTORY` inside `worlds_local/` or backups will nest inside each other.
- **Server browser:** set `SERVER_PUBLIC=true` to list in Steam's server browser (requires real UDP forwarding for 2456-2458 if NAT). For LAN/direct IP join, players enter `host:2457` (gameport+1) in the in-game address field.
- **Crossplay:** `CROSSPLAY=true` switches matchmaking to PlayFab (accepts Xbox/MS Store) and uses UDP 2458. Some mods don't work with crossplay — it defaults to `false`.
- **Admin:** admins are listed in `/config/adminlist.txt` (`data/config/adminlist.txt`) or via `ADMINLIST_IDS`.
- The upstream repo also exposes `SUPERVISOR_HTTP`/`STATUS_HTTP` web endpoints (ports 9001/TCP, 80/TCP) and a `sys_nice` capability — the capability is included here; the web servers are off by default and not port-mapped.
