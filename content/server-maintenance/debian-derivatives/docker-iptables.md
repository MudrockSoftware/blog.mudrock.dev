---
title: Docker iptables
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [docker, firewall, debian]
---

Docker adds iptables rules that skip your firewall. Every published port is open to the world. Turn it off.

## Disable

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'JSON'
{
  "iptables": false,
  "ip6tables": false
}
JSON
sudo systemctl restart docker
sudo reboot   # flushes old Docker chains
```

If `daemon.json` already exists, merge the keys instead of overwriting.

## Restore outbound internet for containers

Without Docker's rules, containers have no NAT.

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```

Edit `/etc/default/ufw`:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Add to the top of `/etc/ufw/before.rules`, before `*filter`:

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 172.16.0.0/12 ! -o docker0 -j MASQUERADE
COMMIT
```

```bash
sudo ufw reload
```

## Test

```bash
docker run --rm -p 8080:80 nginx
curl -m 3 http://<public-ip>:8080        # from another machine: must time out
docker run --rm alpine ping -c1 1.1.1.1  # must work
```

## Alternative: bind to localhost

Keep Docker defaults and publish only on loopback. [[server-maintenance/caddy|Caddy]] serves them publicly.

```yaml
ports:
  - "127.0.0.1:8080:80"
```

Rootless Podman avoids this problem: [[server-maintenance/podman-compose|Podman Compose]].
