---
title: ufw
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [firewall, debian]
---

Allow SSH before enabling, or you lock yourself out.

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp   # HTTP/3
sudo ufw enable
sudo ufw status verbose
```

Allow from one IP:

```bash
sudo ufw allow from 203.0.113.10 to any port 5432 proto tcp
```

Allow Tailscale ([[vpn/tailscale/install|Tailscale]]):

```bash
sudo ufw allow in on tailscale0
```

Delete a rule:

```bash
sudo ufw status numbered
sudo ufw delete <number>
```

Docker bypasses ufw. Published ports stay open to the internet. Fix: [[server-maintenance/debian-derivatives/docker-iptables|Docker iptables]].
