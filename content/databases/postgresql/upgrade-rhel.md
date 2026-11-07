---
title: Upgrade (RHEL)
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [postgres, rhel, almalinux]
---

Example: 16 to 17. Change the numbers for your versions. Minor updates (17.1 to 17.2) need only an image or package update.


Assumes the official PGDG repo and `postgresql16-server` already running.

## 1. Install the new version

```bash
sudo dnf install -y postgresql17-server postgresql17-contrib
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
```

Install the same extensions for 17 too (for example `postgis35_17`, `pgvector_17`).

## 2. Back up

```bash
sudo -u postgres pg_dumpall > /root/dump-16.sql
```

## 3. Check

```bash
sudo systemctl stop postgresql-16
cd /tmp
sudo -u postgres /usr/pgsql-17/bin/pg_upgrade \
  -b /usr/pgsql-16/bin -B /usr/pgsql-17/bin \
  -d /var/lib/pgsql/16/data -D /var/lib/pgsql/17/data \
  --check
```

Fix anything it reports. Run again until `*Clusters are compatible*`.

## 4. Upgrade

```bash
sudo -u postgres /usr/pgsql-17/bin/pg_upgrade \
  -b /usr/pgsql-16/bin -B /usr/pgsql-17/bin \
  -d /var/lib/pgsql/16/data -D /var/lib/pgsql/17/data \
  --clone    # XFS/Btrfs reflink: fast and keeps 16 intact. Use --copy otherwise.
```

Do not use `--link` if you want rollback. With `--link`, the old cluster is unusable once the new server starts.

## 5. Move config and start

```bash
sudo diff /var/lib/pgsql/16/data/postgresql.conf /var/lib/pgsql/17/data/postgresql.conf
sudo cp /var/lib/pgsql/16/data/pg_hba.conf /var/lib/pgsql/17/data/pg_hba.conf
# copy your custom settings into 17/data/postgresql.conf

sudo systemctl disable postgresql-16
sudo systemctl enable --now postgresql-17
sudo -u postgres psql -c 'SELECT version();'
sudo -u postgres /usr/pgsql-17/bin/vacuumdb --all --analyze-in-stages
```

## Rollback

```bash
sudo systemctl disable --now postgresql-17
sudo systemctl enable --now postgresql-16
```

With `--clone` or `--copy`, the 16 cluster is unchanged. If it fails to start, restore from the dump:

```bash
sudo -u postgres psql -f /root/dump-16.sql postgres
```

## Cleanup (after a few days)

```bash
sudo -u postgres ./delete_old_cluster.sh   # created by pg_upgrade in /tmp
sudo dnf remove -y postgresql16\*
```

## AppStream module (no PGDG)

```bash
sudo systemctl stop postgresql
sudo cp -a /var/lib/pgsql/data /var/lib/pgsql/data.bak
sudo dnf module reset -y postgresql
sudo dnf module enable -y postgresql:16
sudo dnf install -y postgresql-upgrade
sudo postgresql-setup --upgrade
sudo systemctl start postgresql
```

Rollback: stop, `dnf module reset` and enable the old stream, `dnf distro-sync`, then restore `data.bak` to `data`.

Postgres in Docker: [[databases/postgresql/upgrade-docker|Upgrade (Docker)]]. Debian/Ubuntu: [[databases/postgresql/upgrade-debian|Upgrade (Debian)]].
