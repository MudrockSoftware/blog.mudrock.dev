---
title: Upgrade (Docker)
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [postgres, docker]
---

Example: 16 to 17. Change the numbers for your versions. Minor updates (17.1 to 17.2) need only an image or package update.


Safest method. The old volume is never touched, so rollback is instant.

Compose before:

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata16:/var/lib/postgresql/data
```

## 1. Back up

```bash
docker compose stop app            # stop writers
docker compose exec -T db pg_dumpall -U postgres > dump-16.sql
ls -lh dump-16.sql
```

## 2. Switch to new version with a new volume

```yaml
services:
  db:
    image: postgres:17
    volumes:
      - pgdata17:/var/lib/postgresql/data
volumes:
  pgdata16:
  pgdata17:
```

Keep `pgdata16` in the file so compose does not forget it.

## 3. Restore

```bash
docker compose up -d db
until docker compose exec db pg_isready -U postgres; do sleep 1; done
docker compose exec -T db psql -U postgres -v ON_ERROR_STOP=1 < dump-16.sql
docker compose exec db psql -U postgres -c 'SELECT version();'
docker compose exec db vacuumdb -U postgres --all --analyze-in-stages
docker compose up -d
```

Since PostgreSQL 14, `password_encryption` defaults to `scram-sha-256`. If old roles used md5, the dump keeps md5 hashes. Reset passwords if clients fail to log in.

## Rollback

```bash
docker compose down
# revert image to postgres:16 and volume to pgdata16
docker compose up -d
```

Delete the old volume only after a few days: `docker volume rm <project>_pgdata16`.

## Large databases: pg_upgrade in Docker

Dumps are slow for hundreds of GB. Use the `pgautoupgrade/pgautoupgrade` image, or `tianon/postgres-upgrade:16-to-17` with both data dirs mounted:

```bash
docker compose stop db
docker run --rm \
  -v <project>_pgdata16:/var/lib/postgresql/16/data \
  -v <project>_pgdata17:/var/lib/postgresql/17/data \
  tianon/postgres-upgrade:16-to-17
```

This copies data (no `--link`), so `pgdata16` stays valid for rollback. Copy `pg_hba.conf` changes over by hand.


Native installs: [[databases/postgresql/upgrade-debian|Upgrade (Debian)]], [[databases/postgresql/upgrade-rhel|Upgrade (RHEL)]].
