---
title: Install
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [tailscale, vpn, linux]
---

## Install

Any distro:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale up --ssh
```

Open the printed URL to log in. Headless servers: create an auth key in the admin console, then:

```bash
sudo tailscale up --auth-key=tskey-auth-XXXX --ssh
```

Check:

```bash
tailscale status
tailscale ip -4
```

## Kernel settings for subnet router or exit node

Needed only if the node forwards traffic for others.

```bash
cat <<'CONF' | sudo tee /etc/sysctl.d/99-tailscale.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
CONF
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise:

```bash
sudo tailscale set --advertise-exit-node
sudo tailscale set --advertise-routes=192.168.1.0/24
```

Approve both in the admin console (Machines, Edit route settings).
To skip manual approval, use `autoApprovers` in [[vpn/tailscale/access-control|Access Control]].

On firewalld, enable masquerade so forwarded traffic gets out:

```bash
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```

## UDP throughput (Linux 6.2+)

Improves exit node and subnet router speed.

```bash
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
sudo ethtool -K "$NETDEV" rx-udp-gro-forwarding on rx-gro-list off
```

Persist it with a NetworkManager dispatcher script:

```bash
sudo tee /etc/NetworkManager/dispatcher.d/pre-up.d/50-tailscale >/dev/null <<'SH'
#!/bin/sh
ethtool -K "$(ip -o route get 8.8.8.8 | cut -f 5 -d ' ')" rx-udp-gro-forwarding on rx-gro-list off
SH
sudo chmod 755 /etc/NetworkManager/dispatcher.d/pre-up.d/50-tailscale
```

On Debian without NetworkManager, use a oneshot systemd unit with the same command.

## Firewall

Tailscale works through NAT with no open ports. For direct connections, allow UDP 41641:

```bash
sudo firewall-cmd --permanent --add-port=41641/udp
sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0
sudo firewall-cmd --reload
# ufw:
sudo ufw allow 41641/udp
sudo ufw allow in on tailscale0
```

Trusting `tailscale0` opens every port to your tailnet. Restrict it with [[vpn/tailscale/access-control|Access Control]].

## Disable key expiry for servers

Admin console, Machines, server, Disable key expiry. Otherwise the node drops off after 180 days.

Tagged machines have key expiry disabled automatically. See [[vpn/tailscale/access-control|Access Control]].
