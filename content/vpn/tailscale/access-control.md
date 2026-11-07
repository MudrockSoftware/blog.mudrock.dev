---
title: Access Control
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [tailscale, vpn, security]
---

Group machines with tags. Then allow only the ports each group needs. Everything else is blocked.

Open the policy file: [Access controls](https://login.tailscale.com/admin/acls) in the admin console. Switch to the **JSON editor**.

## Concepts

- **Tag** (`tag:server`): a machine group. A tagged machine belongs to the tailnet, not to a user.
- **Group** (`group:admins`): a list of users.
- **Grant**: who (`src`) can reach what (`dst`) on which ports (`ip`).
- Nothing in grants means blocked. The default policy allows everything. Delete that rule.

## Example policy

Machines:

- `tag:server`: all servers
- `tag:web`: public web servers (Caddy)
- `tag:db`: database servers

```json
{
  "groups": {
    "group:admins": ["mudrock@github"]
  },

  "tagOwners": {
    "tag:server": ["group:admins"],
    "tag:web":    ["group:admins"],
    "tag:db":     ["group:admins"]
  },

  "grants": [
    // admins: everything on every server
    { "src": ["group:admins"], "dst": ["tag:server"], "ip": ["*"] },

    // web servers: only Postgres on db servers
    { "src": ["tag:web"], "dst": ["tag:db"], "ip": ["tcp:5432"] },

    // all tailnet members: only HTTP/HTTPS on web servers
    { "src": ["autogroup:member"], "dst": ["tag:web"], "ip": ["tcp:80", "tcp:443"] }
  ],

  "ssh": [
    {
      "action": "accept",
      "src":    ["group:admins"],
      "dst":    ["tag:server"],
      "users":  ["mudrock", "root"]
    }
  ],

  "autoApprovers": {
    "routes":   { "192.168.1.0/24": ["tag:server"] },
    "exitNode": ["tag:server"]
  },

  "tests": [
    { "src": "tag:web",          "accept": ["tag:db:5432"], "deny": ["tag:db:22"] },
    { "src": "mudrock@github",   "accept": ["tag:db:22"] }
  ]
}
```

Replace `mudrock@github` with your login. It is shown under **Users** in the admin console.

Click **Save**. The editor runs `tests` and refuses to save if one fails.

## Port syntax

| `ip` value | Meaning |
|---|---|
| `"*"` | all ports, all protocols |
| `"tcp:22"` | one TCP port |
| `"udp:53"` | one UDP port |
| `"tcp:8000-8100"` | port range |
| `"tcp:*"` | all TCP ports |
| `"icmp:*"` | ping |

## Tag machines

Every tag a machine uses must be in `tagOwners` first.

From the admin console: **Machines**, `...` menu on the machine, **Edit ACL tags**.

From the machine:

```bash
sudo tailscale set --advertise-tags=tag:server,tag:db
```

New machines: create an auth key with tags (**Settings**, **Keys**, **Generate auth key**, **Tags**). Then:

```bash
sudo tailscale up --auth-key=tskey-auth-XXXX --ssh
```

Tagged machines never expire their key. No need to disable key expiry by hand.

## Add a new group of machines

1. Add `"tag:newgroup": ["group:admins"]` to `tagOwners`.
2. Add a grant with the ports it needs.
3. Save, then tag the machines.

## Check from a machine

```bash
tailscale status                       # shows tags per machine
nc -zv <tailscale-ip> 5432             # allowed port: succeeded
nc -zv -w 3 <tailscale-ip> 22          # blocked port: times out
```

Rules apply on top of the host firewall. Trusting `tailscale0` in firewalld or ufw is safe once grants are tight.
