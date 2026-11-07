---
title: Caddy
author: Ufuk Furkan Öztürk
date: 2026-11-07
tags: [caddy, reverse-proxy, tls]
---

Layout:

```
/etc/caddy/
├── Caddyfile
├── hosts/
│   └── example-com.conf
└── snippets/
    ├── cloudflare-tls.conf
    ├── ipfilter.conf
    └── logs.conf
```

One file per site. Shared blocks live in snippets.

## Install with the Cloudflare DNS module

Stock Caddy has no DNS providers. Add the module:

```bash
# AlmaLinux / Fedora
sudo dnf install -y 'dnf-command(copr)'
sudo dnf copr enable -y @caddy/caddy
sudo dnf install -y caddy
# Debian / Ubuntu: see https://caddyserver.com/docs/install#debian-ubuntu-raspbian

sudo caddy add-package github.com/caddy-dns/cloudflare
caddy list-modules | grep cloudflare
```

`add-package` replaces the binary. A package update brings back the stock binary, so run it again after updates. Or pin with `sudo dnf versionlock add caddy`.

## Cloudflare token

Create an API token with `Zone.DNS:Edit` for your zone. Keep it out of the Caddyfile:

```bash
sudo systemctl edit caddy
```

```ini
[Service]
Environment=CF_API_TOKEN=your-token
```

```bash
sudo systemctl restart caddy
```

## Caddyfile

```caddyfile
{
	email admin@example.com
	servers {
		protocols h1 h2 h3
		trusted_proxies static private_ranges
	}
}

import snippets/*.conf
import hosts/*.conf
```

Snippets load before hosts, so hosts can `import` them.

## snippets/logs.conf

```caddyfile
(logs) {
	log {
		output file /var/log/caddy/{args[0]}.log {
			roll_size 50MiB
			roll_keep 5
			roll_keep_for 720h
		}
		format json
	}
}
```

## snippets/cloudflare-tls.conf

```caddyfile
(cloudflare-tls) {
	tls {
		dns cloudflare {env.CF_API_TOKEN}
		resolvers 1.1.1.1
	}
}
```

DNS challenge works behind Cloudflare proxy, for wildcards, and on hosts with port 80 closed.

## snippets/ipfilter.conf

Allow only listed ranges. Everyone else gets 403.

```caddyfile
(ipfilter) {
	@blocked not remote_ip 192.168.1.0/24 100.64.0.0/10
	respond @blocked 403
}
```

`100.64.0.0/10` is the Tailscale range (see [[vpn/tailscale/install|Tailscale]]). Behind Cloudflare proxy, use `client_ip` instead of `remote_ip` and add Cloudflare ranges to `trusted_proxies`.

## Shared security headers

Add `snippets/headers.conf`:

```caddyfile
(headers) {
	encode zstd gzip
	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains"
		X-Content-Type-Options nosniff
		Referrer-Policy strict-origin-when-cross-origin
		-Server
	}
}
```

## hosts/example-com.conf

```caddyfile
example.com, www.example.com {
	import cloudflare-tls
	import logs example-com
	import headers

	@www host www.example.com
	redir @www https://example.com{uri} permanent

	reverse_proxy 127.0.0.1:8080
}

*.example.com {
	import cloudflare-tls
	import logs wildcard-example-com

	@admin host admin.example.com
	handle @admin {
		import ipfilter
		reverse_proxy 127.0.0.1:9000
	}

	handle {
		respond 404
	}
}
```

Bind backends to `127.0.0.1` so only Caddy reaches them. See Docker iptables ([[server-maintenance/debian-derivatives/docker-iptables|Debian]], [[server-maintenance/rhel-derivatives/docker-iptables|RHEL]]).

## Validate and reload

```bash
sudo mkdir -p /var/log/caddy && sudo chown caddy:caddy /var/log/caddy
caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
journalctl -u caddy -f
```

`caddy fmt` formats only the main file. Format host files with `caddy fmt --overwrite /etc/caddy/hosts/example-com.conf`.

## Firewall

```bash
sudo firewall-cmd --permanent --add-service=http --add-service=https --add-port=443/udp
sudo firewall-cmd --reload
# ufw:
sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp
```

UDP 443 enables HTTP/3. See [[server-maintenance/debian-derivatives/ufw|ufw]] or [[server-maintenance/rhel-derivatives/firewalld|firewalld]].

## SELinux (AlmaLinux)

If Caddy returns 502 to local backends:

```bash
sudo setsebool -P httpd_can_network_connect 1
```
