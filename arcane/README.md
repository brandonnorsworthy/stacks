# arcane

Arcane Docker manager. Web UI on host port `10000` (container 3552, Arcane's native port), manages Docker via the host socket.

- Image: `ghcr.io/getarcaneapp/manager:v2.15.1` (pinned)
- Port: host `10000` → container `3552` (recorded per the managed allocation scheme; first claimed slot)
- UID/GID: `10001:65534` (dedicated stack UID / nogroup). Arcane's image entrypoint starts as root to prepare then drops to `PUID:PGID`, so `PUID=10001` is what enforces the dedicated UID (setting compose `user:` would skip the entrypoint).

## First-time setup

1. Generate the encryption key (32-byte hex) and create `.env`:

   ```sh
   cp .env.example .env
   sed -i "s|replace-with-openssl-rand-hex-32|$(openssl rand -hex 32)|" .env
   ```

2. Set real values in `.env` (`APP_URL`, and `PROJECTS_DIRECTORY` if you want Arcane to manage this repo's stacks).

3. Own the data volume as the stack UID:

    ```sh
    docker run --rm -v arcane:/d busybox chown -R 10001:65534 /d
    ```

    (On a local/dev machine, skip the next step until you're ready to try the socket.)

4. Allow the stack UID to use the Docker socket (socket is root:docker 0660). Recommended: create a stub user at that UID and add it to the docker group (survives socket recreation):

    ```sh
    sudo useradd -u 10001 -g nogroup -M -d /nonexistent -s /usr/sbin/nologin arcane
    sudo usermod -aG docker arcane
    ```

    Alternative: grant an ACL directly (requires `sudo apt install acl`):

    ```sh
    sudo setfacl -m u:10001:rw /var/run/docker.sock
    ```

    (ACLs persist across socket recreation only if Docker's daemon is configured to keep the socket; re-run after a Docker daemon reinstall/upgrade.)

5. Start (on the server the repo lives at `/srv/stacks`; on a local/dev machine use `compose.dev.yml`, which remounts the project at the local clone path and overrides `PROJECTS_DIRECTORY`):

    ```sh
    # server
    docker compose up -d

    # local / dev
    docker compose -f compose.yml -f compose.dev.yml up -d
    ```

## Notes

- `cgroup: host` is required for container-ID detection / self-upgrades.
- **Self-upgrade drift:** Arcane rewrites its own image when it self-upgrades, which moves the DB schema past the pinned tag in this file. The next `docker compose up` then runs the *older* pinned image against a *newer* DB and crash-loops (`database schema version N is newer than this Arcane binary supports`). Fix: bump the `image:` tag (and this line) to the version Arcane actually upgraded to, then re-run — no data loss, the volume holds the schema.
- `ENCRYPTION_KEY` is stored in `.env` (untracked); losing it locks the Arcane database.
- Named volume `arcane` (declared `external: true`) holds the Arcane DB and project metadata — back it up.
- Project management requires matched absolute paths, hence the `/srv/stacks:/srv/stacks` mount + `PROJECTS_DIRECTORY=/srv/stacks` (overridden by `compose.dev.yml` on local/dev machines).
- **Security:** Arcane's `PUID` only governs files *Arcane itself* creates — the stacks it manages run as whatever their own compose files say. Because Arcane holds the Docker socket (root-equivalent), `PUID` scoping does not protect Arcane itself: keep host port `10000` off the public internet (VPN/Tailscale only) and guard `ENCRYPTION_KEY`.
