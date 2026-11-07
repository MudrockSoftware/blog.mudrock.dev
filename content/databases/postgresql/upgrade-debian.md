---
title: Upgrade (Debian)
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [postgres, debian, ubuntu]
---

Example: 16 to 17. Change the numbers for your versions. Minor updates (17.1 to 17.2) need only `apt upgrade`.

Debian and Ubuntu use `postgresql-common`. It runs several versions side by side and upgrades with `pg_upgradecluster`.

## 1. Add the PGDG repo (if the new version is not in your distro)

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

## 2. Install the new version

```bash
sudo apt install -y postgresql-17
pg_lsclusters
```

Install the same extensions for 17 too (for example `postgresql-17-postgis-3`, `postgresql-17-pgvector`).

The package creates an empty `17 main` cluster. Drop it, or the upgrade fails:

```bash
sudo pg_dropcluster 17 main --stop
```

## 3. Back up

```bash
sudo -u postgres pg_dumpall > /root/dump-16.sql
```

## 4. Upgrade

```bash
sudo systemctl stop postgresql   # stop writers first
sudo pg_upgradecluster -m upgrade 16 main
pg_lsclusters
```

`-m upgrade` uses `pg_upgrade` with copy mode. The 16 cluster stays intact. Do not add `--link` if you want rollback.

What it does:

- Creates `17 main` on port 5432 with your config and `pg_hba.conf`.
- Moves `16 main` to port 5433 and stops it.

## 5. Start and check

```bash
sudo systemctl start postgresql
sudo -u postgres psql -c 'SELECT version();'
sudo -u postgres vacuumdb --all --analyze-in-stages
```

## Rollback

```bash
sudo pg_dropcluster 17 main --stop
sudo sed -i 's/^port = .*/port = 5432/' /etc/postgresql/16/main/postgresql.conf
echo auto | sudo tee /etc/postgresql/16/main/start.conf
sudo systemctl restart postgresql
pg_lsclusters
```

If 16 fails to start, restore from the dump:

```bash
sudo -u postgres psql -f /root/dump-16.sql postgres
```

## Cleanup (after a few days)

```bash
sudo pg_dropcluster 16 main
sudo apt purge -y postgresql-16 postgresql-client-16
```

Postgres in Docker: [[databases/postgresql/upgrade-docker|Upgrade (Docker)]]. AlmaLinux: [[databases/postgresql/upgrade-rhel|Upgrade (RHEL)]].
