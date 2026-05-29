# pihole

DNS server, DHCP, ad/tracker blocking, and internal name resolution.

## Overview

Pi-hole provides DNS and DHCP for the local network, and is the mechanism by which `*.home.philippthesurfer.com` resolves to the Mini PC's LAN IP for devices on the network. It runs with `network_mode: host` so it can bind to port 53 directly and support DHCP broadcasts — as a result it is **not** routed through Traefik. The Ansible role also adds a custom DNS entry for `auth.philippthesurfer.com` (Keycloak's public domain) so it resolves internally without a round-trip through public DNS.

## Prerequisites

- `common` role applied (Docker)
- Port 53 (TCP + UDP) open on UFW — added by the `common` role

## Containers

| Container | Image | Purpose |
|---|---|---|
| `pihole` | `pihole/pihole:2026.05.0` | DNS + DHCP + ad blocking |

Pi-hole runs with `network_mode: host`. It is not on the `proxy` Docker network and has no Traefik labels.

## Configuration

Files rendered by Ansible to the host:

| File | Purpose |
|---|---|
| `/opt/homelab/services/pihole/docker-compose.yml` | Stack definition |
| `/opt/homelab/services/pihole/etc-pihole/custom.list` | Custom DNS overrides (internal hostnames) |

The `custom.list` file maps every `*.home.philippthesurfer.com` service hostname to `192.168.178.20`, plus `auth.philippthesurfer.com`. Additional DNS records can be added manually in the Pi-hole admin UI or appended to `custom.list`.

## Secrets

All secrets live in HashiCorp Vault at `secret/data/ansible`.

| Vault field | Variable | Purpose |
|---|---|---|
| `pihole_webpassword` | `vault_pihole_webpassword` | Web admin UI password |

## URL

`http://192.168.178.20:8080`

Pi-hole is not behind Traefik. It is accessible directly by LAN IP on port 8080. No HTTPS for the admin UI.

## Deploy

```bash
ansible-playbook ansible/site.yml --tags pihole
```

## Notes

**Adding DNS records for new services:** Either add them manually in the Pi-hole admin UI under *Local DNS → DNS Records*, or append to `/opt/homelab/services/pihole/etc-pihole/custom.list` and restart:

```bash
echo "192.168.178.20 newservice.home.philippthesurfer.com" >> /opt/homelab/services/pihole/etc-pihole/custom.list
docker restart pihole
```

**DHCP:** If Pi-hole is used for DHCP, disable the FritzBox's built-in DHCP server first to avoid conflicts.

**Upstream DNS:** Pi-hole forwards non-local queries upstream (default: `1.1.1.1`, `8.8.8.8`). Configure upstream resolvers in the Pi-hole admin UI under *Settings → DNS*.
