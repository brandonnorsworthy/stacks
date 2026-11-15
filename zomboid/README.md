# zomboid

Project Zomboid (Build 42) dedicated server for playing in first person with Project Viewpoint.

- Image: `danixu86/project-zomboid-dedicated-server:42.21-release-2` (Docker Hub, pinned). The tag is only the image build; the game itself is installed into `./data/pz-dedicated` and is at **42.22.2** (see Game version).
- Ports: UDP `16261` (game) and `16262` (direct connection). These are Project Zomboid's well-known ports, so nothing is claimed from the `10000+` pool. Both come from `PORT` / `UDPPORT` in `.env`, and the server listens on the same numbers inside the container. The dev PC uses `16263` / `16264` because Brandon's own game hosts on the default pair there. RCON (27015) is not published.
- UID/GID: `10011:65534` (dedicated stack UID; `10010` = spokes). The image's entrypoint has to start as root (it chowns the data folders and runs steamcmd) and then drops to its `steam` user, which is UID 1000 in the image and not configurable. The `entrypoint:` wrapper in `compose.yml` renumbers `steam` to `10011:65534` before calling the image's own entrypoint, so the game server process runs as 10011. Do not re-own `/home/steam` itself: steamcmd then fails with "Couldn't create cache directory ... error 13".
- Limits: `mem_limit: 16g`, `cpus: 6.0` (game-server tier). Java heap 2 to 8 GB (`MIN_MEMORY` / `MAX_MEMORY`).
- Data (all under `./data`, gitignored):
  - `./data/Zomboid` → `/home/steam/Zomboid`: `Server/` (ini, SandboxVars, spawn files), `Saves/Multiplayer/<SERVERNAME>/` (the world), `db/<SERVERNAME>.db` (accounts), logs.
  - `./data/pz-dedicated` → `/home/steam/pz-dedicated`: the game install and downloaded Workshop mods. Files here are owned by root (steamcmd), the workshop folder by 10011.

## Mods

| Workshop ID | Mod | Mod ID |
|---|---|---|
| 3619862853 | ZombieBuddy | `ZombieBuddy` |
| 3809306528 | Project Viewpoint | `Viewpoint` |
| 3810302175 | 3D models for Viewpoint | `PZVoxelStudioViewpoint` |

Viewpoint and the 3D models are client-side. The server lists the mods so every client loads the same set; it does not run the ZombieBuddy Java agent. Each player needs ZombieBuddy installed in their own game (launch option `-javaagent:ZombieBuddy.jar --` on Linux/macOS, `-agentlib:zbNative --` on Windows).

## Game version

Project Viewpoint only works on the exact game build it was made for (`versionMin=42.22`, `versionMax=42.22`), and clients must match the server's build.

- `FORCEUPDATE=true` runs steamcmd at start and moves the install to the current public build. Use it once on a new install (`data/pz-dedicated` starts empty, about 7 GB download), then set it to `false`.
- With `false`, the install in `data/pz-dedicated` never changes, so a game update on Steam does not break the server. Update on purpose, when the mods support the new build.
- Steam only serves the current build. If the public branch has moved on, a new install cannot get the older build; copy `data/pz-dedicated` from a machine that has it.

## First-time setup

1. `cp .env.example .env`, set `PASSWORD` and `ADMINPASSWORD`, set `FORCEUPDATE=true`.
2. Create `data/Zomboid/Server` and `data/pz-dedicated`. Optionally put a `<SERVERNAME>_SandboxVars.lua` in `data/Zomboid/Server` to control the world settings.
3. Start it from Arcane (Up). Wait for `*** SERVER STARTED ****` in the log.
4. Set `FORCEUPDATE=false`.

Join from the game with Join → IP and port, account name of your choice, and the server password.

## Moving a world in

The world is three things named after `SERVERNAME`. With the stack stopped, and after backing up what is there:

1. `Saves/Multiplayer/<SERVERNAME>/` → `data/Zomboid/Saves/Multiplayer/`
2. `Server/<SERVERNAME>_SandboxVars.lua`, `_spawnregions.lua`, `_spawnpoints.lua` → `data/Zomboid/Server/` (keep the existing `.ini`: it holds the passwords, ports and mod list)
3. `db/<SERVERNAME>.db` → `data/Zomboid/db/`

Then `chown -R 10011:65534` what was copied (the entrypoint also does this for `data/Zomboid` on start) and start the stack. Characters are tied to the account name, so players log in with the name they used before.
