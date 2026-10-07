# Migrating existing server deployments into stacks

Guide for moving something that is **already running on the server** (an ad-hoc compose file, a repo that builds its own image on the box, a hand-run container) into this repo. For brand-new stacks, skip this and use [templates/](templates/README.md).

## Principles

1. **Deploy only pulls and runs.** A stack's `compose.yml` references a published image (`image:`), never `build:`. No `Dockerfile`, build context, or `docker build` on the server. Personal apps are built by a GitHub Action and pushed to Docker Hub / GHCR (see [AGENTS.md](AGENTS.md)); the stack just pulls the pinned tag.
2. **Runtime behavior is preserved; shape is normalized.** What the container *does* today (image, env, ports, volumes, networks, restart/stop behavior, resource needs) is the source of truth for the migrated stack. How the compose file *looks* follows [templates/README.md](templates/README.md). Where the two conflict, the migrated stack must still work first, then be brought as close to the template as possible, and every remaining difference is documented (see "Deviations").
3. **Nothing secret lands in git.** Existing `.env` values stay on the server. Only variable names go into `.env.example`, with `changeme` placeholders.
4. **No SSH.** Agents never log into the server to inspect it. The user supplies the existing compose file (and `docker inspect`/`docker stats` output if useful) and the agent works from that.

## What to keep vs. what to normalize

| Aspect | Rule when migrating |
| --- | --- |
| `image` | Keep the same image, but **pin** it to a specific version/digest. Use the version actually running now (ask for `docker inspect`/`docker ps` output if the compose says `:latest` or only a major like `postgres:16`). Replace `build:` with a published image (see "Replacing `build:`"). |
| `container_name` | Keep if other things reference it (e.g. DB hostnames in other apps' env). Must stay unique on the server. |
| `env_file` / `environment` | Keep the same variables. Secrets go to `.env` (server only), names to `.env.example`. Inline secrets in the old compose are a security finding: report them immediately and move them to `.env`. |
| `ports` | Keep existing host ports so clients and firewall rules keep working. Record them in the stack README. New ports follow the allocation scheme in the template; well-known ports (e.g. `5432`, `25565`) are fine as-is. |
| `volumes` | Keep the existing data (see "Preserving data"). Template rule is `./data:/...`; named volumes are fine for databases. |
| `networks` | Keep existing external networks that other stacks depend on. Template allows `networks:` only for the database or LLM network, so any other custom network is a deviation to question. |
| `restart`, `stop_grace_period` | Keep existing values when they are deliberate (e.g. a long grace period so a database can flush). Otherwise use `unless-stopped` / `30s`. |
| `mem_limit`, `cpus` | Start from what is set today, then **round to a template tier** (`0.5/2g`, `1/4g`, `3/8g`, `6/16g`, `12/32g`). If there are no limits today, pick the tier from the service type. If real usage clearly doesn't fit the tier, note it as a deviation. |
| `user` | Add the dedicated stack UID (`<next UID>:65534`, recorded in the stack README). See "Things that usually need adding". |
| `healthcheck` | Add one if the image supports it. |

## Process

1. **Gather the current state** from the user: the existing compose file(s), any `.env` variable *names* (not values), image tags actually running, and the data locations (bind mount paths / named volumes).
2. **Create the stack folder** from [templates/](templates/) (`compose.yml`, `.env.example`, `README.md`), claiming the next UID per [AGENTS.md](AGENTS.md).
3. **Port the runtime settings** from the old compose into the new one using the table above.
4. **Normalize the shape** (limits tier, pinned image, healthcheck, `user`, data dir).
5. **Write the stack README**: image, ports, UID, limits, data locations, and a "Migration notes" section listing what changed from the old deployment and any deviations with the reason.
6. **Plan the cutover** (below) and hand it to the user. Don't assume the old container can be stopped without asking.
7. Commit once the user confirms the migrated stack runs correctly on the server.

## Replacing `build:`

If the old compose or repo builds on the server:

1. Make sure the project repo has a GitHub Action that builds and pushes the image to GHCR or Docker Hub (the repo only needs the Dockerfile and the workflow).
2. Reference the pushed image in the stack with a pinned tag (a version or `sha-<short>` tag, not `latest`).
3. Remove `build:`, `context:`, and bind mounts of source code from the compose. Anything the app needs at runtime comes from env, volumes, or the image.
4. Check the Dockerfile for baked-in env/secrets. If any exist, report them immediately and move them to `.env`.

## Preserving data

Data survival is the most important part of a migration.

- **Named volumes** (e.g. `postgres_data`): keep the same volume name in the new compose so the existing data is reused. If the volume belongs to the old compose project, it is prefixed with that project name; check with `docker volume ls` (the user runs it) and use `external: true` with the real name, or copy the data across.
- **Bind mounts**: either keep pointing at the existing host path, or move the data into the stack's `./data` and `chown` it to the new stack UID, while the old container is stopped.
- **Databases**: stop the old container cleanly (honor its grace period) before copying or re-mounting, or take a dump first.
- Never delete the old volume or directory until the user has confirmed the new stack works. Keep it as the rollback.

## Cutover and rollback

1. Take a backup or dump of anything stateful.
2. Stop the old deployment (`docker compose stop`, not `down -v`).
3. Deploy the stack through Arcane.
4. Verify: container healthy, dependent apps can still reach it, data present.
5. Rollback is starting the old compose again; this is why the old data stays untouched until verification.

Two stacks can't bind the same host port or `container_name`, so the old one must be stopped before the new one starts.

## Things that usually need adding

- **Dedicated UID** (`user: "<uid>:65534"`). Existing data must be owned by that UID, so plan a `chown`. Some images (e.g. Postgres) support `user:` but need the data dir to be owned by it; others start as root and drop privileges via `PUID`/`PGID` env instead (see the [valheim README](valheim/README.md)). Record whichever approach is used.
- **Healthcheck** if the image has a suitable command.
- **Pinned image version** instead of a floating tag.
- **Resource tier** rounded to the allowed values.

## Deviations

Anything that can't match the template (a non-tier limit, an extra network, a root-started container, a floating tag you can't pin yet) goes in the stack README under "Deviations" with the reason, so it can be revisited later.

## Worked example: shared Postgres

Existing server compose:

```yaml
services:
  postgres:
    container_name: postgres
    image: postgres:16
    env_file: .env
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    stop_grace_period: 120s
    mem_limit: 4g
    cpus: 1
    networks:
      - shared-postgres

volumes:
  postgres_data:

networks:
  shared-postgres:
    external: true
```

Review against the guide:

| Setting | Existing | Migrated | Why |
| --- | --- | --- | --- |
| `container_name: postgres` | ok | keep | Other apps likely connect by this hostname. |
| `image: postgres:16` | floating minor | pin, e.g. `postgres:16.<minor>` | Pin to the version actually running (check `docker ps`) so the on-disk data format matches. Never jump major versions as part of a migration. |
| `env_file: .env` | ok | keep | Variable names (`POSTGRES_USER`, `POSTGRES_PASSWORD`, ...) go in `.env.example` with placeholders. |
| `ports: 5432:5432` | ok | keep | Well-known port, and existing clients depend on it. |
| `postgres_data` named volume | ok | keep | Named volumes are allowed for databases. Keep the same name (`external: true` if it is prefixed by the old project) so the data carries over. |
| `restart: unless-stopped` | ok | keep | |
| `stop_grace_period: 120s` | deliberate | keep | Lets Postgres shut down cleanly; longer than the template's `30s` on purpose. |
| `mem_limit: 4g` / `cpus: 1` | matches the `1 cpu / 4gb` tier | keep | Already on a template tier. |
| `networks: shared-postgres` (external) | ok | keep | The template permits a database network; other stacks join this one. |
| `user` | missing | add dedicated UID | Requires the data to be owned by that UID. For a named volume this means `chown`ing the volume contents, or running as the image's `postgres` user and recording that as a deviation. |
| `healthcheck` | missing | add `pg_isready` | The image ships the tool. |

Resulting `compose.yml` (UID is a placeholder, claim the real next one):

```yaml
services:
  postgres:
    container_name: postgres
    image: postgres:16.4                 # pin to the version currently running
    env_file: .env
    user: "10003:65534"                  # dedicated stack UID; data must be chowned to it
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
    restart: unless-stopped
    stop_grace_period: 120s
    mem_limit: 4g
    cpus: 1
    networks:
      - shared-postgres

volumes:
  postgres_data:

networks:
  shared-postgres:
    external: true
```

Cutover for this one: take a `pg_dumpall` first, stop the old container with its full 120s grace, deploy the stack, confirm `pg_isready` passes and dependent apps reconnect, then leave the old volume untouched until verified.
