# Nextcloud

## Overview

Nextcloud is a self-hosted productivity platform that provides file storage, sharing, and collaboration tools.

## Key Features

- File sync and sharing
- Calendar, contacts, and collaboration apps
- End-to-end encryption

## Configuration

Nextcloud is deployed using the All-in-One (AIO) container. The web interface is available at `https://office.home.com`.

## Usage

- Access Nextcloud at [https://office.home.com](https://office.home.com)
- Install additional apps as needed
- Manage the AIO stack (updates, backups, stop/start) at `https://<host>:8999`

## Common Tasks

All Nextcloud admin commands go through `occ`, run as `www-data` inside the `nextcloud-aio-nextcloud` container. To save typing, define an alias in your shell:

```bash
alias occ='docker exec -it --user www-data nextcloud-aio-nextcloud php occ'
```

The commands below use this alias. Without it, replace `occ` with the full `docker exec …` line above.

> If a command fails with "No such container", check the name with `docker ps | grep nextcloud`.

### Passwords & Login

There are two separate passwords:

| Password | Used for | How to recover |
|---|---|---|
| Nextcloud user password | Logging in at `office.home.com` | Reset with `occ user:resetpassword` |
| AIO passphrase | The AIO interface on port 8999 | Read it back from the mastercontainer (it can't be changed) |

```bash
# Reset a user's password (prompts twice)
occ user:resetpassword <username>

# Show the AIO interface passphrase
docker exec nextcloud-aio-mastercontainer grep password /mnt/docker-aio-config/data/configuration.json

# Locked out by two-factor auth → disable 2FA for a user
occ twofactorauth:disable <username> totp

# Too many failed logins (brute-force throttling) → reset for your IP
occ security:bruteforce:reset <ip-address>
```

The initial `admin` password is also shown in the AIO interface, as long as you never changed it.

### Users & Groups

```bash
occ user:list                                     # list users
occ user:add --display-name="Jane" --group="family" jane   # create user (prompts for password)
occ user:disable <username>                       # block login, keep data
occ user:enable <username>
occ user:delete <username>                        # delete user and their files
occ group:adduser admin <username>                # make someone admin
occ user:info <username>                          # quota, groups, last login
occ user:setting <username> files quota "50 GB"   # set a quota
```

### Files

```bash
# Files copied directly into the data dir don't show up → rescan
occ files:scan <username>          # one user
occ files:scan --all               # everyone (slow)

# "File is locked" errors → AIO stores locks in Redis; restarting it clears them
docker restart nextcloud-aio-redis

# Empty a user's trash / old file versions to free space
occ trashbin:cleanup <username>
occ versions:cleanup <username>
```

The data directory on the host is `NEXTCLOUD_DATADIR` (default `./volumes/nextcloud/data`).

### Apps

```bash
occ app:list                 # installed apps (enabled / disabled)
occ app:install <app>        # e.g. calendar, contacts, deck, notes
occ app:enable <app>
occ app:disable <app>        # disable a broken app that prevents login
occ app:update --all
```

### Maintenance & Health

```bash
occ status                         # version, maintenance mode, installed?
occ maintenance:mode --on          # put in maintenance mode
occ maintenance:mode --off         # stuck on "in maintenance mode" page → turn it off
occ db:add-missing-indices         # fix "missing indices" warning in admin overview
occ maintenance:repair --include-expensive   # fix mimetype / repair warnings
occ setupchecks                    # same checks as Administration → Overview
```

### Logs

```bash
docker logs -f nextcloud-aio-nextcloud      # PHP / container logs
docker logs -f nextcloud-aio-apache         # web server
docker logs -f nextcloud-aio-mastercontainer
occ log:tail                                # Nextcloud application log
```

Or open the containers in [Dozzle](dozzle.md) at `https://monitor.home.com`.

### Updates

Nextcloud AIO manages its own containers, so `docker compose pull` only updates the mastercontainer (see [Updating Images](../README.md#updating-images)).

To update Nextcloud itself:

1. Open the AIO interface at `https://<host>:8999`.
2. Click **Stop containers**.
3. Click **Start and update containers**.

Don't run `occ upgrade` or the web updater yourself, because AIO handles that.

### Backups

AIO has a built-in backup (BorgBackup). Set it up once in the AIO interface under **Backup and restore**, choosing the target directory. Then:

- **Create a backup now**: AIO interface → **Create backup**. Containers are stopped during the backup.
- **Daily automatic backups**: enable them in the same section, with an optional automatic update after the backup.
- **Restore**: the same section → **Restore selected backup**.

Keep the **backup encryption password** shown by AIO somewhere safe (e.g. a password manager). Without it the backups can't be restored.

### Start / Stop the Whole Stack

Use the AIO interface (**Stop containers** / **Start containers**). Stopping only the mastercontainer with `docker compose stop` leaves the Nextcloud containers running.

## Links & Resources

- [Official Documentation](https://docs.nextcloud.com/)
- [GitHub Repository](https://github.com/nextcloud/all-in-one)

