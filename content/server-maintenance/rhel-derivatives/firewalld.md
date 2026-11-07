---
title: firewalld
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [firewall, rhel]
---

```bash
sudo dnf install -y firewalld
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=443/udp   # HTTP/3
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Allow a port from one IP:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.10/32" port port="5432" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Trust Tailscale ([[vpn/tailscale/install|Tailscale]]):

```bash
sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0
sudo firewall-cmd --reload
```

Remove a rule:

```bash
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
```

Docker bypasses firewalld. Published ports stay open to the internet. Fix: [[server-maintenance/rhel-derivatives/docker-iptables|Docker iptables]].
