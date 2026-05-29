# common

Base host configuration — Docker, UFW firewall, deploy user, and shared Docker network.

## Overview

This role runs first and prepares the Mini PC for all other service roles. It installs Docker CE with the Compose plugin, configures UFW with a strict default-deny policy (only SSH, HTTP, HTTPS, and DNS are open), adds the deploy user to the `docker` group, creates the base directory structure under `/opt/homelab/services`, and creates the shared `proxy` Docker network that every service stack joins.

## Prerequisites

- Debian-based host with `apt`
- SSH access as root or a user with sudo privileges
- The deploy user (`philipp`) must already exist on the host

## What It Installs

| Package / Resource | Purpose |
|---|---|
| `docker-ce`, `docker-ce-cli`, `containerd.io` | Docker engine |
| `docker-buildx-plugin`, `docker-compose-plugin` | Build and Compose support |
| `ufw` | Host firewall |
| Docker network `proxy` | Shared network — all Traefik-routed containers attach here |
| Directory `/opt/homelab/services` | Base path for all service stacks |

## Firewall Rules

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH |
| 80 | TCP | Traefik HTTP (redirects to HTTPS) |
| 443 | TCP | Traefik HTTPS |
| 53 | TCP + UDP | Pi-hole DNS |

Default policy: incoming **deny**, outgoing **allow**.

## Secrets

None. This role has no secret dependencies.

## Deploy

```bash
ansible-playbook ansible/site.yml --tags common
```

## Notes

The `common` role is idempotent but only needs to run once per host. After first setup it is commented out in `site.yml` to avoid unnecessary overhead on every deploy. Re-enable it if provisioning a fresh host or after an OS reinstall.
