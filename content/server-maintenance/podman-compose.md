---
title: Podman Compose
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [podman, docker, linux]
---

## Overview

For self-updating containers, run each compose service as a Quadlet systemd user unit with `AutoUpdate=registry` and a major-version tag such as `:2`. Podman's built-in updater only touches containers that systemd started, so containers launched with plain `podman compose up` will not update themselves.

| Approach                                                                   | Best for                                              | Self-updating                                         |
| -------------------------------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------------- |
| `podman compose` or `docker compose` on the rootless Podman socket         | Quick start, testing, keeping compose files unchanged | No (only with a pull-and-recreate timer, no rollback) |
| Quadlet units (`.container`, `.network`, `.volume`) under `systemd --user` | Long-running services, start at boot, rootless        | Yes, with automatic rollback                          |

This guide assumes a systemd-based distro and Podman 5.x. Quadlet needs Podman 4.4 or newer; `Notify=healthy` and `.pod` files need Podman 5.

## Prerequisites: rootless setup

Do these once per host. The examples use a dedicated service user called `svc`; your own login user works too.

1. Install Podman and its rootless helpers.
   - Fedora / RHEL: `sudo dnf install podman podman-compose passt`
   - Debian / Ubuntu: `sudo apt install podman podman-compose uidmap passt slirp4netns`
2. Create the user and check it has subordinate UID/GID ranges.

```bash
sudo useradd -m -s /bin/bash svc
grep svc /etc/subuid /etc/subgid
# if nothing is printed:
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 svc
```

3. Enable lingering, so the user's services start at boot and keep running after logout.

```bash
sudo loginctl enable-linger svc
```

4. Log in as the user with a real session. `sudo su - svc` does not set `XDG_RUNTIME_DIR` or the user D-Bus, so `systemctl --user` fails. Use SSH, or:

```bash
sudo machinectl shell svc@
# fallback: sudo -iu svc, then
export XDG_RUNTIME_DIR=/run/user/$(id -u)
```

5. Allow ports below 1024 if a container must publish 80 or 443 directly (otherwise use high ports behind a reverse proxy).

```bash
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-rootless-ports.conf
sudo sysctl --system
```

6. Verify, as `svc`:

```bash
podman system migrate
podman info --format '{{.Host.Security.Rootless}}'   # true
podman run --rm docker.io/library/alpine id
```

Three rules that save trouble later:

- **Fully qualify every image** (`docker.io/library/nginx:1`, not `nginx`). Short names can prompt or fail, and auto-update requires the full name.
- **Bind-mount permissions**: container root is your UID; other container UIDs map into the subuid range. Fix ownership with `podman unshare chown 999:999 ./data`, or use `UserNS=keep-id` (Quadlet) / `userns_mode: keep-id` (compose).
- **SELinux hosts**: add `:Z` to bind mounts (`./data:/data:Z`). Named volumes need nothing.

## Option A: run compose files directly

Your existing `docker-compose.yml` runs unchanged, but updates need a timer you write yourself. Pick one of two front ends.

**podman compose** (Podman's wrapper; it calls `podman-compose` or `docker-compose`, whichever is installed):

```bash
cd ~/stacks/myapp
podman compose up -d
podman compose ps
```

**docker compose against the rootless Podman socket** (best compatibility with complex files):

```bash
systemctl --user enable --now podman.socket
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock' >> ~/.bashrc
source ~/.bashrc
docker compose up -d
```

To bring containers back after a reboot, set `restart: always` on each service and enable Podman's restart service:

```bash
systemctl --user enable podman-restart.service
```

**Updating without Quadlet.** `podman auto-update` skips these containers because systemd did not start them. The workaround is a user timer that pulls and recreates the stack. With `:2`-style tags this gives the same semver behaviour, but nothing rolls back a broken update.

`~/.config/systemd/user/myapp-update.service`:

```ini
[Unit]
Description=Pull and recreate myapp

[Service]
Type=oneshot
WorkingDirectory=%h/stacks/myapp
ExecStart=/usr/bin/podman compose pull
ExecStart=/usr/bin/podman compose up -d
ExecStartPost=/usr/bin/podman image prune -f
```

`~/.config/systemd/user/myapp-update.timer`:

```ini
[Timer]
OnCalendar=*-*-* 04:00
RandomizedDelaySec=15m
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now myapp-update.timer
```

## Option B (recommended): convert compose to Quadlet

Quadlet turns small unit files in `~/.config/containers/systemd/` into systemd user services, one per container. You get boot start, `journalctl` logs, dependency ordering and auto-update for free.

| Compose                         | Quadlet                                           |
| ------------------------------- | ------------------------------------------------- |
| `services.web`                  | `web.container`                                   |
| top-level `networks.x`          | `x.network`                                       |
| named `volumes.x`               | `x.volume`                                        |
| `image`                         | `Image=` (fully qualified)                        |
| `container_name` / service name | `ContainerName=`                                  |
| `ports`                         | `PublishPort=`                                    |
| `environment` / `env_file`      | `Environment=` / `EnvironmentFile=`               |
| `volumes` on a service          | `Volume=` (`x.volume:/path` or `%h/dir:/path:Z`)  |
| `networks` on a service         | `Network=x.network`                               |
| `command`                       | `Exec=`                                           |
| `healthcheck`                   | `HealthCmd=`, `HealthInterval=`, `HealthRetries=` |
| `depends_on`                    | `[Unit]` `Requires=` + `After=` `db.service`      |
| `restart: always`               | `[Service]` `Restart=always`                      |

**Automatic conversion.** [Podlet](https://github.com/containers/podlet) generates Quadlet files from a compose file. Treat its output as a draft and review it:

```bash
podman run --rm -v "$PWD":"$PWD":Z -w "$PWD" quay.io/k9withabone/podlet compose docker-compose.yml
```

**Load and start.** Quadlet units are generated, so you `start` them rather than `enable` them; the `[Install]` section handles boot.

```bash
systemctl --user daemon-reload
systemctl --user start web.service
```

Three details that differ from compose:

- Default names get a `systemd-` prefix. Set `ContainerName=` and `NetworkName=` so containers still find each other by the old service names over DNS.
- If a unit is missing after `daemon-reload`, the file has an error. Check with `/usr/libexec/podman/quadlet -dryrun -user` (on Debian/Ubuntu: `/usr/lib/podman/quadlet`).
- `%h` expands to the user's home directory, so paths stay portable.

## Auto-updates with semver-style tags

Set `AutoUpdate=registry` and use a floating major tag like `:2`; Podman then pulls every new 2.x.y release and never jumps to 3.0. On each run, `podman auto-update` compares the local image digest with the registry digest for the same tag, pulls if they differ, and restarts the unit.

The semver behaviour comes from the image publisher, who moves the `:2` tag to each new 2.x release. Check the image's tag list before relying on it; not every project publishes major or minor tags.

| Tag you set | Updates you get                           | Use when                                 |
| ----------- | ----------------------------------------- | ---------------------------------------- |
| `:2`        | Minor and patch (2.4.1 → 2.5.0)           | Project follows semver strictly          |
| `:2.4`      | Patch only (2.4.1 → 2.4.3)                | Minor releases have broken things before |
| `:2.4.1`    | None (only security rebuilds of that tag) | You want manual control                  |
| `:latest`   | Everything, including breaking majors     | Avoid for databases and stateful apps    |

**1. Mark each container** in its `.container` file:

```ini
[Container]
Image=docker.io/library/postgres:16
AutoUpdate=registry
```

Use `AutoUpdate=local` instead for images you build yourself; it restarts the unit when the local image changes.

**2. Enable the timer** (runs daily around midnight by default):

```bash
systemctl --user enable --now podman-auto-update.timer
systemctl --user list-timers
```

To change the schedule, run `systemctl --user edit podman-auto-update.timer` and add:

```ini
[Timer]
OnCalendar=
OnCalendar=*-*-* 04:00
RandomizedDelaySec=15m
```

**3. Test it** without changing anything:

```bash
podman auto-update --dry-run
```

The `UPDATED` column shows `pending` for containers with a newer image waiting.

**4. Make rollback meaningful.** Auto-update rolls back to the previous image when the restarted unit fails. By default "fails" only means "process crashed on start". Add a health check and `Notify=healthy` so systemd waits for the container to report healthy, and a bad update gets reverted:

```ini
[Container]
HealthCmd=pg_isready -U postgres
HealthInterval=30s
HealthRetries=5
Notify=healthy
```

**5. Private registries.** `podman login` writes credentials to `$XDG_RUNTIME_DIR`, which is wiped at reboot. Store them persistently instead:

```bash
podman login --authfile ~/.config/containers/auth.json registry.example.com
```

**6. Clean old images** with a drop-in (`systemctl --user edit podman-auto-update.service`):

```ini
[Service]
ExecStartPost=/usr/bin/podman image prune -f
```

## When floating tags are not enough

Podman only follows the tag you wrote; it never reads version numbers or resolves ranges like `^2.3`. If an image publishes only exact tags (`2.4.1`, `2.4.2`), `:2` does not exist and you need a tool that rewrites the tag for you.

The usual answer is [Renovate](https://docs.renovatebot.com/): keep your Quadlet or compose files in a git repo, pin exact versions, and let Renovate open pull requests for new versions within the range you allow. A `customManagers` regex entry can match `Image=` lines in `.container` files. After merging, the host runs `git pull`, `systemctl --user daemon-reload` and restarts the changed units (a small timer can do this).

```json
{
  "customManagers": [
    {
      "customType": "regex",
      "fileMatch": ["\\.container$"],
      "matchStrings": ["Image=(?<depName>[^:\\s]+):(?<currentValue>[^\\s@]+)"],
      "datasourceTemplate": "docker"
    }
  ],
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "enabled": false
    }
  ]
}
```

The `packageRules` entry blocks major upgrades, which reproduces "stay on 2.x" for images without a `:2` tag. Watchtower-style tools add nothing here: they also only re-pull the same tag, which `podman auto-update` already does with rollback.

## Worked example: WordPress + MariaDB

This converts a typical two-service compose file into five Quadlet files that start at boot and update within their major versions. Confirm the `:11` and `:6` tags exist on Docker Hub before deploying.

**The original compose file**

```yaml
services:
  db:
    image: docker.io/library/mariadb:11
    env_file: db.env
    volumes: ["db-data:/var/lib/mysql"]
    restart: always
  wordpress:
    image: docker.io/library/wordpress:6
    depends_on: [db]
    ports: ["8080:80"]
    env_file: wp.env
    volumes: ["wp-data:/var/www/html"]
    restart: always
volumes:
  db-data: {}
  wp-data: {}
```

**Secrets** in `~/.config/wp/` (then `chmod 600 ~/.config/wp/*.env`):

```bash
# db.env
MARIADB_DATABASE=wp
MARIADB_USER=wp
MARIADB_PASSWORD=change-me
MARIADB_RANDOM_ROOT_PASSWORD=1

# wp.env
WORDPRESS_DB_HOST=db
WORDPRESS_DB_USER=wp
WORDPRESS_DB_PASSWORD=change-me
WORDPRESS_DB_NAME=wp
```

**Quadlet files** in `~/.config/containers/systemd/`:

`wp.network`

```ini
[Network]
NetworkName=wp
```

`db-data.volume` and `wp-data.volume` (one file each, same content)

```ini
[Volume]
```

`db.container`

```ini
[Unit]
Description=WordPress database

[Container]
ContainerName=db
Image=docker.io/library/mariadb:11
AutoUpdate=registry
Network=wp.network
Volume=db-data.volume:/var/lib/mysql
EnvironmentFile=%h/.config/wp/db.env
HealthCmd=healthcheck.sh --connect --innodb_initialized
HealthInterval=30s
HealthRetries=5
Notify=healthy

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

`wordpress.container`

```ini
[Unit]
Description=WordPress
Requires=db.service
After=db.service

[Container]
ContainerName=wordpress
Image=docker.io/library/wordpress:6
AutoUpdate=registry
Network=wp.network
PublishPort=8080:80
Volume=wp-data.volume:/var/www/html
EnvironmentFile=%h/.config/wp/wp.env

[Service]
Restart=always

[Install]
WantedBy=default.target
```

**Bring it up and turn on updates**

```bash
systemctl --user daemon-reload
systemctl --user start wordpress.service      # starts db first via Requires=
systemctl --user status db wordpress
systemctl --user enable --now podman-auto-update.timer
podman auto-update --dry-run
```

The site is on `http://<host>:8080`. When MariaDB 12 ships, `:11` keeps you on 11.x, which is what you want: major database upgrades need a manual migration.

## Komodo under an isolated `komodo` user

Run the whole Komodo stack as Quadlet units owned by a locked-down `komodo` user, and hand Periphery that user's rootless Podman socket instead of `/var/run/docker.sock`. If Komodo is compromised, the attacker gets the `komodo` user and its containers, not root and not your other users' data. Komodo [supports Podman through its Docker-compatible API](https://komo.do/docs/intro).

**What changes from Komodo's compose file**

| Compose line                                          | Rootless replacement                                 | Why                                                                                        |
| ----------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `/var/run/docker.sock:/var/run/docker.sock`           | `%t/podman/podman.sock:/var/run/docker.sock`         | Periphery can only control the `komodo` user's containers                                  |
| `/proc:/proc`                                         | Removed                                              | It exposes every host process to the container; Periphery just loses the host process list |
| `/etc/komodo` root directory                          | `%h/komodo/periphery` (same path inside and outside) | The `komodo` user can't write to `/etc`                                                    |
| `image: ...postgres-documentdb` / `ferretdb` (no tag) | Exact pinned tag, no `AutoUpdate`                    | Komodo warns database updates can break                                                    |
| `komodo-core` / `komodo-periphery` `:2`               | `:2` + `AutoUpdate=registry`                         | Updates within Komodo 2.x automatically                                                    |
| `${VAR}` substitution                                 | Literal values in env files                          | Quadlet doesn't do compose-style substitution                                              |
| `9120:9120`                                           | `127.0.0.1:9120:9120`                                | Reachable only through a reverse proxy on the host                                         |

### 1. Create and lock down the user (as `mudrock`)

```bash
sudo useradd -m -s /bin/bash komodo
sudo passwd -l komodo                  # no password login
sudo chmod 700 /home/*                 # komodo can't read other homes
groups komodo                          # must NOT list sudo, wheel or docker
grep komodo /etc/subuid /etc/subgid    # add ranges if empty (Prerequisites step 2)
sudo loginctl enable-linger komodo
sudo machinectl shell komodo@
```

The `docker` group check matters most: if rootful Docker is installed and `komodo` can reach its socket, the isolation is gone, because that socket is root-equivalent.

### 2. Prepare files (as `komodo`)

```bash
systemctl --user enable --now podman.socket podman-restart.service
mkdir -p ~/komodo/backups ~/komodo/periphery ~/.config/komodo ~/.config/containers/systemd
```

Put Komodo's `compose.env` in `~/.config/komodo/compose.env` with `KOMODO_DATABASE_USERNAME` and `KOMODO_DATABASE_PASSWORD` filled in. Then add two more env files with the same credentials and run `chmod 600 ~/.config/komodo/*.env`:

```bash
# ~/.config/komodo/postgres.env
POSTGRES_USER=komodo
POSTGRES_PASSWORD=change-me-long-random
POSTGRES_DB=postgres

# ~/.config/komodo/ferretdb.env
FERRETDB_POSTGRESQL_URL=postgres://komodo:change-me-long-random@postgres:5432/postgres
```

### 3. Quadlet files in `~/.config/containers/systemd/`

`komodo.network`

```ini
[Network]
NetworkName=komodo
```

`postgres-data.volume`, `ferretdb-state.volume`, `komodo-keys.volume` (one file each)

```ini
[Volume]
```

`postgres.container` (copy the current tag from the [postgres-documentdb package page](https://github.com/FerretDB/documentdb/pkgs/container/postgres-documentdb))

```ini
[Container]
ContainerName=postgres
Image=ghcr.io/ferretdb/postgres-documentdb:<pinned-tag>
Label=komodo.skip=
Network=komodo.network
Volume=postgres-data.volume:/var/lib/postgresql/data
EnvironmentFile=%h/.config/komodo/postgres.env

[Service]
Restart=always

[Install]
WantedBy=default.target
```

`ferretdb.container` (tag from the [FerretDB package page](https://github.com/FerretDB/FerretDB/pkgs/container/ferretdb); use the FerretDB version that matches your postgres-documentdb tag)

```ini
[Unit]
Requires=postgres.service
After=postgres.service

[Container]
ContainerName=ferretdb
Image=ghcr.io/ferretdb/ferretdb:<pinned-tag>
Label=komodo.skip=
Network=komodo.network
Volume=ferretdb-state.volume:/state
EnvironmentFile=%h/.config/komodo/ferretdb.env

[Service]
Restart=always

[Install]
WantedBy=default.target
```

`core.container`

```ini
[Unit]
Requires=ferretdb.service
After=ferretdb.service

[Container]
ContainerName=core
Image=ghcr.io/moghtech/komodo-core:2
AutoUpdate=registry
RunInit=true
Network=komodo.network
PublishPort=127.0.0.1:9120:9120
EnvironmentFile=%h/.config/komodo/compose.env
Environment=KOMODO_DATABASE_ADDRESS=ferretdb:27017
Volume=komodo-keys.volume:/config/keys
Volume=%h/komodo/backups:/backups:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

`periphery.container`

```ini
[Unit]
Requires=core.service
After=core.service

[Container]
ContainerName=periphery
Image=ghcr.io/moghtech/komodo-periphery:2
AutoUpdate=registry
RunInit=true
Network=komodo.network
EnvironmentFile=%h/.config/komodo/compose.env
Environment=PERIPHERY_ROOT_DIRECTORY=%h/komodo/periphery
Volume=komodo-keys.volume:/config/keys
Volume=%t/podman/podman.sock:/var/run/docker.sock
Volume=%h/komodo/periphery:%h/komodo/periphery
# needed on SELinux hosts to reach the socket and stack folders
SecurityLabelDisable=true

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Container names match the compose service names, so `ferretdb:27017`, `postgres:5432` and any `core`/`periphery` addresses in `compose.env` keep working.

### 4. Start and enable updates

```bash
systemctl --user daemon-reload
systemctl --user start periphery.service     # pulls in core, ferretdb, postgres
systemctl --user status postgres ferretdb core periphery
systemctl --user enable --now podman-auto-update.timer
podman auto-update --dry-run                 # only core and periphery are listed
```

Expose Komodo through a reverse proxy with TLS that forwards to `127.0.0.1:9120`. After creating your admin account, turn off open sign-up in `compose.env`.

### 5. Stacks you deploy from Komodo

- Keep every stack's files under `~/komodo/periphery`; Periphery can't see anything else.
- Those stacks are created by compose over the socket, not by systemd, so `podman auto-update` ignores them. Use Komodo's own per-stack update polling for them.
- Give stack services `restart: always`, so `podman-restart.service` brings them back after a reboot.
- Stacks can't publish ports below 1024 unless you set `ip_unprivileged_port_start` (Prerequisites step 5).

### What this does and doesn't protect

An attacker who takes over Komodo can run any command as `komodo`: start containers, read its env files and volumes, and use its network access. They can't use `sudo`, read other users' homes, or touch root-owned files, unless they also have a kernel exploit. Keep the kernel and Podman updated, and keep backups outside `/home/komodo`.

## Troubleshooting and day-2 commands

| Symptom                                          | Likely cause                                 | Fix                                                             |
| ------------------------------------------------ | -------------------------------------------- | --------------------------------------------------------------- |
| `Failed to connect to bus`                       | Switched user with `su`                      | Use SSH or `machinectl shell svc@`, or export `XDG_RUNTIME_DIR` |
| Containers stop at logout or don't start at boot | Lingering off                                | `sudo loginctl enable-linger svc`                               |
| Unit not found after `daemon-reload`             | Error in a Quadlet file                      | Run the `quadlet -dryrun -user` check from Option B             |
| auto-update skips a container                    | Not started by systemd, or no `AutoUpdate=`  | Run it as a Quadlet unit with `AutoUpdate=registry`             |
| `short-name resolution` error                    | Image name not fully qualified               | Prefix `docker.io/library/` or the registry                     |
| Permission denied on a bind mount                | UID mapping or SELinux                       | `podman unshare chown`, `UserNS=keep-id`, add `:Z`              |
| Can't publish port 80/443                        | Unprivileged port limit                      | Set `ip_unprivileged_port_start` (Prerequisites step 5)         |
| Containers can't reach each other by name        | Different networks or `systemd-` name prefix | Same `Network=`, set `ContainerName=`                           |
| Private image stops updating after reboot        | Login stored in tmpfs                        | Log in with `--authfile ~/.config/containers/auth.json`         |

Everyday commands:

```bash
systemctl --user status wordpress           # state of one service
journalctl --user -u wordpress -f           # follow logs
podman ps --format '{{.Names}} {{.Image}} {{.Status}}'
podman auto-update --dry-run                # what would update
podman auto-update                          # update now
journalctl --user -u podman-auto-update     # what the last run did
systemctl --user list-timers                # when it runs next
podman image prune -f                       # drop old images
```
