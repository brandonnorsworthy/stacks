# python-sandbox

[`python-sandbox-mcp`](../../python-sandbox-mcp/README.md): an MCP server with one tool, `run_python`, that runs code in a **fresh bubblewrap jail on every call** (no files, variables or processes survive, no network, no host visibility). Registered in LibreChat as `python`.

## Allocations

- Port: none published. LibreChat reaches it as `http://python-sandbox:3001/mcp` over the external `mcp-internal` network.
- UID/GID: `10009:65534`
- Tier: `1` cpu / `4g`, `pids_limit: 256`. Each run is also capped at `SANDBOX_MEMORY_MB` (1024).
- Image: `ghcr.io/brandonnorsworthy/python-sandbox-mcp:0.1.0` (pinned). **Not published yet**: on the dev PC it is tagged locally (`docker build -t ghcr.io/brandonnorsworthy/python-sandbox-mcp:0.1.0 ../../python-sandbox-mcp`). The server can't use this stack until the repo is pushed and the `release` workflow has published the tag.
- Network: external `mcp-internal`, created with `docker network create --internal mcp-internal` (no route out; only LibreChat and this stack join it).

## Safety

- Container: `read_only`, `cap_drop: ALL`, `no-new-privileges`, no Docker socket, no env secrets, internal-only network (so even a jail escape has no egress).
- `seccomp.json` is Docker's default profile plus an allow rule for `unshare`, `setns`, `clone`, `clone3`, `mount`, `umount2`, `pivot_root`, `sethostname`, `setdomainname`, which bubblewrap needs to build the per-call jail. Without capabilities these only work inside the jail's own user namespace. Compose reads the profile from this folder; if Arcane cannot resolve the relative path, give it an absolute one.
- Per-call: new user/pid/net/ipc/uts/cgroup namespaces, read-only `/usr`, size-capped tmpfs `/tmp`, rlimits, SIGKILL of the whole process group on timeout.

## Env

All optional, set in `compose.yml` (no secrets): `SANDBOX_TIMEOUT_SECONDS`, `SANDBOX_MAX_TIMEOUT_SECONDS`, `SANDBOX_MEMORY_MB`, `SANDBOX_MAX_CONCURRENCY`, `ALLOWED_HOSTS`.

## Clients

- LibreChat: `librechat.yaml` -> `python` -> `http://python-sandbox:3001/mcp`; enable with the `MCP Servers` button in the chat input.
- Nothing else can reach it by design (no published port). To call it by hand: `docker exec librechat node ...` (it is the only other container on `mcp-internal`).
