# arcane

Arcane Docker manager. Web UI on port 3552, manages Docker via the host socket.

- Image: `ghcr.io/getarcaneapp/manager:v1.19.4` (pinned)
- UID/GID: `10001:65534` (dedicated stack UID / nogroup). Arcane's image entrypoint starts as root to prepare then drops to `PUID:PGID`, so `PUID=10001` is what enforces the dedicated UID (setting compose `user:` would skip the entrypoint).

## First-time setup

1. Generate the encryption key (32-byte hex) and create `.env`:

   ```sh
   cp .env.example .env
   sed -i "s|replace-with-openssl-rand-hex-32|$(openssl rand -hex 32)|" .env
   ```

2. Set real values in `.env` (`APP_URL`, and `PROJECTS_DIRECTORY` if you want Arcane to manage this repo's stacks).

3. Own the data dir as the stack UID:

   ```sh
   sudo chown 10001:65534 data
   ```

4. Allow the stack UID to use the Docker socket (socket is root:docker 0660):

   ```sh
   sudo setfacl -m u:10001:rw /var/run/docker.sock
   ```

   (ACLs persist across socket recreation only if Docker's daemon is configured to keep the socket; re-run after a Docker daemon reinstall/upgrade.)

5. Start:

   ```sh
   docker compose up -d
   ```

## Notes

- `cgroup: host` is required for container-ID detection / self-upgrades.
- `ENCRYPTION_KEY` is stored in `.env` (untracked); losing it locks the Arcane database.
- `./data/:/app/data` holds the Arcane DB and project metadata — back it up.
- Project management requires matched absolute paths, hence the `/srv/stacks:/srv/stacks` mount + `PROJECTS_DIRECTORY=/srv/stacks`.
- **Security:** Arcane's `PUID` only governs files *Arcane itself* creates — the stacks it manages run as whatever their own compose files say. Because Arcane holds the Docker socket (root-equivalent), `PUID` scoping does not protect Arcane itself: keep port `3552` off the public internet (VPN/Tailscale only) and guard `ENCRYPTION_KEY`.
