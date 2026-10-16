# Dozzle

## Overview

Dozzle is a lightweight, real-time log viewer for Docker containers. It reads logs straight from the Docker socket (nothing is stored on disk) and shows them in a web UI, along with live CPU/memory stats per container.

## Key Features

- Live log streaming for every container
- Search and filter across logs (text or regex)
- Split view to follow several containers side by side
- Per-container CPU / memory usage
- Download logs

## Configuration

Dozzle is defined in `docker-compose.yml` (service `dozzle`) and exposed through Traefik at `https://monitor.home.com`.

| Setting | Value | Meaning |
|---|---|---|
| `DOZZLE_LEVEL` | `info` | Dozzle's own log level |
| `DOZZLE_TAILSIZE` | `300` | Number of past lines loaded when opening a container |
| `DOZZLE_FILTER` | `status=running` | Only list running containers (remove to also see stopped/crashed ones) |
| Volume | `/var/run/docker.sock:ro` | Read-only access to the Docker API |

> **Security:** there is no authentication in front of Dozzle — anyone who can reach `monitor.home.com` can read every container's logs (which may contain secrets). Keep it LAN-only, or put it behind Authentik forward-auth.

## Usage

Open [https://monitor.home.com](https://monitor.home.com).

- **Follow a service**: click a container in the left sidebar; logs stream live and auto-scroll. Scroll up to pause, click the arrow at the bottom to resume.
- **Search**: press `Ctrl/Cmd + F` inside a log view. Wrap the query in `/.../` for regex (e.g. `/error|fatal/i`).
- **Jump to a container**: press `Ctrl/Cmd + K` and type its name.
- **Compare containers**: use the *pin* icon next to a container to open it in split view (handy for `immich-server` + `immich-postgres`, or `authentik-server` + `authentik-worker`).
- **Stats**: the header of each log view shows live CPU and memory usage.
- **Download**: the menu (⋮) on a log view lets you download the logs or clear the view.

### Typical uses

- After an update (see [Updating Images](../README.md#updating-images)), open the updated container to confirm it started cleanly.
- A service is unreachable through Traefik → check `reverse-proxy` logs for routing/certificate errors, then the service itself.
- A container keeps restarting → temporarily remove `DOZZLE_FILTER` (or use `docker compose logs <service>`) since restarting/stopped containers are hidden by the current filter.

### CLI equivalent

```bash
docker compose logs -f --tail 300 <service>
```

## Links & Resources

- [Official Documentation](https://dozzle.dev/)
- [GitHub Repository](https://github.com/amir20/dozzle)
