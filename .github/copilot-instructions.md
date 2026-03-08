# Copilot Instructions – mini-guide

## Project Overview

This repo is the **backing store for a live WikiJS instance** serving a homelab ProxmoxVE compute cluster. Files here sync directly to/from WikiJS — the deliverable is accurate, runnable documentation. There is no build system or test suite. Content types:
- **Markdown guides** (`.md`, `editor: markdown`) – step-by-step deployment guides for Docker, Kubernetes, and LXC
- **HTML pages** (`.html`, `editor: ckeditor`) – rich-content reference pages (e.g., `servers.html` game server comparison tables) edited in WikiJS's visual editor

## Target Infrastructure

- **Cluster nodes**: `sierra` (Z2-g1a, Strix Halo AI MAX+ 395, 128GB, 10.0.42.38), `tango` (10.0.42.39), `victor` (10.0.42.45), `xray` (10.0.42.46) — HP Z2 workstations, `.ank.com` domain, subnet `10.0.42.x`; see [home.md](../home.md) for current inventory
- **Kubernetes**: minikube with `--driver=none` on `tamlyn.ank.com` (10.0.42.12); MetalLB for LoadBalancer IPs; Avahi/mDNS for `.local` service discovery
- **LXC**: Privileged containers on ProxmoxVE when hardware passthrough (Bluetooth, GPU) is needed
- **mDNS**: Avahi advertiser (`Services/kube/mgk-avahi-advertiser.md`) auto-creates `.local` names for deployed services

## File Naming Conventions

| Prefix | Directory     | Meaning                        |
|--------|---------------|--------------------------------|
| `mgd-` | `Services/docker/` | Mini Guide – Docker       |
| `mgk-` | `Services/kube/`   | Mini Guide – Kubernetes   |
| `lxc-` | `Services/lxc/`    | LXC container guide       |

New guides **must** follow these prefixes and be added to [Services/Mini-Guides.md](../Services/Mini-Guides.md).

## Required Frontmatter (WikiJS format)

Every `.md` file must start with this YAML block:

```yaml
---
title: <filename-without-extension>
description: 
published: true
date: <ISO-8601 datetime>Z
tags: 
editor: markdown
dateCreated: <ISO-8601 datetime>Z
---
```

Dates use the format `2026-03-02T00:00:00.000Z`. Do not omit any field; WikiJS will reject pages without valid frontmatter.

**HTML pages** (ckeditor) embed the same fields in an HTML comment instead of YAML:
```html
<!--
title: Game Servers
description: 
published: true
date: 2025-10-29T03:36:29.739Z
tags: 
editor: ckeditor
dateCreated: 2025-10-26T03:37:53.613Z
-->
```
Do not convert `.html` files to `.md` — ckeditor and markdown editor produce structurally different output.

## Guide Structure Pattern

All guides follow this consistent section structure:

1. **Overview block** – what it does and key features (emoji `📋`)
2. **⚙️ Configuration Variables** (when relevant) – export bash vars at the top so the entire guide is copy-paste ready
3. **✅ Step N:** sections – numbered, sequential, each with a shell command block and a verification step
4. **Troubleshooting** section at the end

Use `✅` for steps, `⚙️` for config, `📋` for overviews, `🔧` for setup parts. See [Services/kube/mgk-wikijs.md](../Services/kube/mgk-wikijs.md) for a canonical example.

## Kubernetes Guide Conventions

- Always create a dedicated namespace per service; set it as the current context
- Use `kubectl apply -f - <<EOF ... EOF` heredocs for inline manifests (no external YAML files)
- Every service gets a PVC for persistent storage; use `ReadWriteOnce`
- Prefer `LoadBalancer` (MetalLB) over `NodePort` to avoid port conflicts; see [mgk-homeassistant.md](../Services/kube/mgk-homeassistant.md) for Bluetooth/privileged access patterns
- Include a verification step after every `kubectl apply`

## LXC Guide Conventions

- Bluetooth/GPU passthrough requires **privileged containers** (`Unprivileged: NO`)
- Bluetooth guides reference the `hassio-bluetti-bt` component in the repo root for device configuration
- AMD GPU guides target the HP Z2 Mini G1A with Proxmox 9.1; see [lxc-llama-amd.md](../Services/lxc/lxc-llama-amd.md)

## Key Reference Files

- [home.md](../home.md) – cluster inventory (nodes, VMs, IPs)
- [Services/Mini-Guides.md](../Services/Mini-Guides.md) – guide index (update when adding guides)
- [Services/kube/mgk-avahi-advertiser.md](../Services/kube/mgk-avahi-advertiser.md) – mDNS/Avahi integration pattern
- [Services/kube/mgk-step-ca.md](../Services/kube/mgk-step-ca.md) – TLS/cert-manager with step-ca (uses `$CA_HOSTNAME` variable pattern)
