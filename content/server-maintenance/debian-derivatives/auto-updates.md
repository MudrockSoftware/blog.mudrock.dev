---
title: Auto Updates
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [updates, debian]
---

Uses `unattended-upgrades`.

```bash
sudo apt update
sudo apt install -y unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades   # answer Yes
```

This writes `/etc/apt/apt.conf.d/20auto-upgrades`:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

Optional, in `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

Defaults cover security updates only. For all updates, uncomment the `-updates` line in `Origins-Pattern` (Debian) or `Allowed-Origins` (Ubuntu).

## Test

```bash
sudo unattended-upgrade --dry-run --debug
systemctl list-timers 'apt-daily*'
cat /var/log/unattended-upgrades/unattended-upgrades.log
[ -f /var/run/reboot-required ] && echo reboot needed
```
