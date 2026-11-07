---
title: SSH User
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [ssh, security, rhel]
---

## Create the user

```bash
useradd -m -G wheel mudrock
```

No prompts. On RHEL, `adduser` is only a symlink to `useradd`, so both work the same.

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
restorecon -R /home/mudrock/.ssh   # SELinux labels
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
sudo sshd -t && sudo systemctl reload sshd
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|allowusers'
```

The `10-` prefix sorts before cloud-init's `50-cloud-init.conf`. First match wins.

## Optional: lock root password

```bash
sudo passwd -l root
```
