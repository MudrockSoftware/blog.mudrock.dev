---
title: Auto Updates
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [updates, rhel]
---

Uses `dnf-automatic`.

```bash
sudo dnf install -y dnf-automatic
```

Edit `/etc/dnf/automatic.conf` (EL 8/9):

```ini
[commands]
upgrade_type = security   # or: default (all updates)
apply_updates = yes
reboot = when-needed      # EL 9.x+: never | when-changed | when-needed
```

```bash
sudo systemctl enable --now dnf-automatic.timer
systemctl list-timers 'dnf-automatic*'
```

## dnf5 (EL 10, Fedora 41+)

Config path differs. Copy defaults first, then edit the same keys:

```bash
sudo cp /usr/share/dnf5/dnf5-plugins/automatic.conf /etc/dnf/dnf5-plugins/automatic.conf
```

## Test

```bash
sudo dnf-automatic --installupdates
journalctl -u dnf-automatic.service
sudo dnf needs-restarting -r
```
