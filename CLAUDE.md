# Blog content rules

Quartz site. Content lives in `content/`. Build and preview: `npx quartz build --serve` (http://localhost:8080).

## Language and style

- Write all posts in English.
- Placeholder admin/login user is always `mudrock` (e.g. `useradd -m -G wheel mudrock`, `ssh mudrock@server`). These guides are used as-is on our servers. Service users with a purpose (e.g. `komodo`, `postgres`, `caddy`) keep their own names.
- Pick the easiest command that works on current releases: one line, no interactive prompts, same shape on every distro where possible. If a newer tool or flag does the job more simply, use it and note the minimum version only if older releases are still common.
  - Example: `useradd -m -s /bin/bash -G sudo mudrock` (Debian) and `useradd -m -G wheel mudrock` (RHEL), not Debian's interactive `adduser`.
- Command-first: show the command, move on. Explain only when a step needs a non-default setting or has a trap.

## Titles

- Keep `title` short: 1-4 words. Readers scan the explorer and the title must fit.
  - Good: `Auto Updates`, `ufw`, `Postgres Upgrade (Docker)`.
  - Bad: `Automatic Security Updates (unattended-upgrades and dnf-automatic)`.
- Put detail in the body, not the title.

## Frontmatter

Every post:

```yaml
---
title: Short Title
author: Ufuk Furkan Öztürk
date: YYYY-MM-DD
tags: [topic, distro]
---
```

## Categories

Categories are folders. Readers must reach every post from the home page in a few clicks.

```
content/
├── index.md                       # links every top-level category
├── server-maintenance/
│   ├── index.md                   # links subcategories + "Any distro" posts
│   ├── <cross-distro posts>.md    # caddy, podman-compose
│   ├── debian-derivatives/        # Debian, Ubuntu (apt, ufw)
│   │   └── index.md
│   └── rhel-derivatives/          # AlmaLinux, Rocky, Fedora, RHEL (dnf, firewalld)
│       └── index.md
├── databases/
│   ├── index.md                   # links every database
│   └── postgresql/                # one folder per database engine
│       ├── index.md
│       ├── upgrade-debian.md
│       ├── upgrade-docker.md
│       └── upgrade-rhel.md
└── vpn/
    ├── index.md                   # links every VPN tool
    └── tailscale/                 # one folder per tool
        ├── index.md
        ├── install.md
        └── access-control.md
```

- Every folder has an `index.md` with a short `title` and a list linking every post in it.
- Adding a post: put it in the right folder and add it to that folder's `index.md`.
- Adding a category: create the folder with `index.md`, link it from the parent `index.md` and `content/index.md`.
- Distro-specific commands go in the distro folder. If a guide differs per distro, write one page per distro with the same file name.
- Cross-distro posts go in the category root.
- Database posts go in `databases/<engine>/`, not in server-maintenance. New engine: new folder with `index.md`, linked from `databases/index.md` and `content/index.md`.
- VPN posts go in `vpn/<tool>/`. Same rules as databases.
- Inside an engine or tool folder, titles drop the engine name (`Upgrade (Docker)`, not `Postgres Upgrade (Docker)`). The folder already says it.

## Branding and llms.txt

- Publisher: MUDROCK SOFTWARE LTD, https://mudrock.dev/. The site is a public secure infrastructure guide and a showcase of the stable infrastructure we run ourselves. We do not sell infrastructure services; never write sales or "hire us" copy. Keep the home page mudrock.dev link and footer link.
- `content/llms.txt` is served at `/llms.txt` for AI agents. When adding, moving or renaming a post or category, update it too. Use absolute `https://blog.mudrock.dev/...` URLs.
- No hidden text or hidden instructions for AI. Attribution stays visible and honest.

## Links

- Use full-path wikilinks with a label: `[[server-maintenance/rhel-derivatives/firewalld|firewalld]]`. Distro folders share file names, so short links are ambiguous.
