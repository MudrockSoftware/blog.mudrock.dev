---
title: SSH User
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [ssh, security, debian]
---

## Create the user

```bash
useradd -m -s /bin/bash -G sudo mudrock
```

One line, no prompts. Use `useradd`, not `adduser`: on Debian `adduser` is an interactive wrapper that asks for a password and contact details. Without `-s /bin/bash`, Debian's `useradd` gives the user `/bin/sh`.

No password is set. Login works only with the SSH key below.

## Passwordless sudo

```bash
echo 'mudrock ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/mudrock
chmod 440 /etc/sudoers.d/mudrock
visudo -c
```

## Copy the SSH key

```bash
install -d -m 700 -o mudrock -g mudrock /home/mudrock/.ssh
install -m 600 -o mudrock -g mudrock /root/.ssh/authorized_keys /home/mudrock/.ssh/authorized_keys
```

Or from your machine: `ssh-copy-id mudrock@server`.

## Test from a new terminal

Keep the root session open until this works.

```bash
ssh mudrock@server
sudo whoami   # root, no password prompt
```

## Lock down sshd

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'CONF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers mudrock
CONF
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|allowusers'
```

The `10-` prefix sorts before cloud-init's `50-cloud-init.conf`. First match wins.

## Optional: lock root password

```bash
sudo passwd -l root
```
