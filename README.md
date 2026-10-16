# Homelab Project
---

See each service's documentation for setup, configuration, and usage details.

## Getting Started

### Media & Knowledge Management
- [Mealie (Recipe Manager)](docs/mealie.md)
- [Booklore (Book Tracking)](docs/booklore.md)
- [Immich (Photo Management)](docs/immich.md)

### Storage & Collaboration
- [Nextcloud](docs/nextcloud.md)

### Home Automation
- [Home Assistant](docs/homeassistant.md)

### Networking & Security
- [Authentik (Identity Provider)](docs/authentik.md)
- [Pihole (DNS Filtering)](docs/pihole.md)
- [Traefik (Reverse Proxy)](docs/traefik.md)

### Monitoring
- [Dozzle (Container Logs)](docs/dozzle.md)

This repository contains the configuration and documentation for a self-hosted homelab environment, organized by domain of functionality.

## Updating Images

Run every command from the repository root on the host. `docker compose up -d` only recreates containers whose image (or config) changed, so it is safe to run on a live stack.

### Update everything (floating tags only)

```bash
docker compose pull && docker compose up -d && docker image prune -f
```

This picks up new versions for every service on a floating tag (`latest`, `stable`, `release`, `alpine`, …). Pinned services (see [below](#services-with-pinned-versions)) are **not** upgraded by this.

Check the result:

```bash
docker compose ps
docker compose logs -f <service>   # or use Dozzle: https://monitor.home.com
```

### Services on `latest` / floating tags

Copy-paste the line for the service you want to update.

| Service | Image | Update command |
|---|---|---|
| Pi-hole | `pihole/pihole:latest` | `docker compose pull pihole && docker compose up -d pihole` |
| Home Assistant | `home-assistant:stable` | `docker compose pull homeassistant && docker compose up -d homeassistant` |
| Immich (server + ML) | `immich-*:release` | `docker compose pull immich-server immich-machine-learning && docker compose up -d immich-server immich-machine-learning` |
| Immich Redis | `redis:7-alpine` | `docker compose pull immich-redis && docker compose up -d immich-redis` |
| Authentik Redis | `redis:alpine` | `docker compose pull authentik-redis && docker compose up -d authentik-redis` |
| Booklore | `booklore/booklore:latest` | `docker compose pull booklore && docker compose up -d booklore` |
| Mealie | `mealie:latest` | `docker compose pull mealie && docker compose up -d mealie` |
| Dozzle | `amir20/dozzle:latest` | `docker compose pull dozzle && docker compose up -d dozzle` |
| Nextcloud AIO (mastercontainer) | `all-in-one:latest` | `docker compose pull nextcloud-aio-mastercontainer && docker compose up -d nextcloud-aio-mastercontainer` |

Notes:
- **Immich**: always update `immich-server` and `immich-machine-learning` together, and read the [release notes](https://github.com/immich-app/immich/releases) first — breaking changes are announced there.
- **Nextcloud AIO**: the command above only updates the mastercontainer. Nextcloud itself (and its child containers) is updated from the AIO interface at `https://<host>:8999` → *Stop containers* → *Start and update containers*.

### Services with pinned versions

These services use a fixed version tag, so `docker compose pull` will never upgrade them. To upgrade, **change the tag first**, then pull and recreate.

#### Traefik — `traefik:v3.6`

Edit `docker-compose.yml` (`reverse-proxy` service) and bump the tag (e.g. `traefik:v3.7`). Check the [migration guide](https://doc.traefik.io/traefik/migration/v3/) for minor-version changes.

```bash
docker compose pull reverse-proxy && docker compose up -d reverse-proxy
```

> Patch releases of `v3.6` are picked up by a plain `pull` since `v3.6` floats over `v3.6.x`.

#### Authentik — `ghcr.io/goauthentik/server:${AUTHENTIK_TAG:-2025.12.1}`

Set the new version in `.env` (rather than editing the compose default):

```bash
echo "AUTHENTIK_TAG=2026.2.1" >> .env   # replace with the target version
```

```bash
docker compose pull authentik-server authentik-worker && docker compose up -d authentik-server authentik-worker
```

- Server and worker must always run the **same** version.
- Upgrade one `YYYY.MM` release at a time (don't skip releases) and read the [release notes](https://docs.goauthentik.io/docs/releases).
- Back up the database first (see below).

#### Authentik Postgres — `postgres:17-alpine`

Minor versions (17.x) update with a plain pull:

```bash
docker compose pull authentik-db && docker compose up -d authentik-db
```

A **major** upgrade (e.g. 17 → 18) cannot reuse the data volume; dump, recreate, restore:

```bash
docker compose stop authentik-server authentik-worker
docker compose exec authentik-db pg_dumpall -U authentik > authentik-backup.sql
docker compose down authentik-db
docker volume rm homelab_database            # check the name with `docker volume ls`
# edit docker-compose.yml → image: postgres:18-alpine
docker compose up -d authentik-db
docker compose exec -T authentik-db psql -U authentik -d authentik < authentik-backup.sql
docker compose up -d authentik-server authentik-worker
```

#### Immich Postgres — `ghcr.io/immich-app/postgres:17-vectorchord0.5.3-pgvector0.8.1`

Only change this tag when an Immich release explicitly asks for it (they publish the exact tag in the release notes / their `docker-compose.yml`). Back up first:

```bash
docker compose exec immich-postgres pg_dumpall -U immich > immich-backup.sql
# edit docker-compose.yml → new immich-postgres tag
docker compose pull immich-postgres && docker compose up -d immich-postgres
```

#### Booklore MariaDB — `lscr.io/linuxserver/mariadb:11.4.5`

Back up, bump the tag in `docker-compose.yml` (staying on the 11.4 LTS line is the safest), then recreate. MariaDB upgrades the data files in place on startup.

```bash
docker compose exec booklore-db sh -c 'mariadb-dump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases' > booklore-backup.sql
# edit docker-compose.yml → e.g. lscr.io/linuxserver/mariadb:11.4.8
docker compose pull booklore-db && docker compose up -d booklore-db
```


