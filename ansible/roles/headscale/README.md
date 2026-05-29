# headscale

Self-hosted Tailscale VPN control plane with a web admin UI.

## Overview

Headscale replaces the Tailscale SaaS control plane, giving full ownership over the mesh VPN. It coordinates device authentication, key exchange, and ACLs for the Tailscale mesh network — actual data plane traffic flows peer-to-peer and never touches this server. Headplane is a web UI layered on top for managing nodes, users, and pre-auth keys without the CLI. Both services sit behind Traefik; the Headscale control plane additionally needs to be reachable from the public internet via `vpn.philippthesurfer.com` for remote device registration. A Postgres database stores node state.

## Prerequisites

- `common` role applied (Docker, `proxy` network)
- `traefik` role running (TLS + DDNS for `vpn.philippthesurfer.com`)
- `keycloak` role running (OIDC for Headplane)
- FritzBox port forwarding: UDP 3478 → `192.168.178.20` (STUN for NAT traversal)

## Containers

| Container | Image | Purpose |
|---|---|---|
| `headscale` | `headscale/headscale:latest` | VPN control plane |
| `headplane` | `ghcr.io/tale/headplane:latest` | Web admin UI |
| `headscale-db` | `postgres:18-alpine` | State database |

Networks: `headscale` and `headplane` join `proxy` (Traefik) and `headscale_internal` (database). The database is not externally routable.

## Configuration

Files rendered by Ansible to the host:

| File | Purpose |
|---|---|
| `/opt/homelab/services/headscale/docker-compose.yml` | Stack definition |
| `/opt/homelab/services/headscale/config/config.yaml` | Headscale server configuration |
| `/opt/homelab/services/headscale/headplane.yaml` | Headplane configuration |

## Secrets

All secrets live in HashiCorp Vault at `secret/data/ansible`.

| Vault field | Variable | Purpose |
|---|---|---|
| `headscale_db_password` | `vault_headscale_db_password` | Postgres password |
| `headscale_oidc_secret` | `vault_headscale_oidc_secret` | Keycloak OIDC client secret for Headscale |
| `headplane_api_key` | `vault_headplane_api_key` | Headscale API key for Headplane to manage nodes |
| `headplane_cookie_secret` | `vault_headplane_cookie_secret` | Session cookie signing secret for Headplane |

## URLs

| Service | URL | Access |
|---|---|---|
| Headscale control plane | `https://vpn.philippthesurfer.com` | Public (required for remote node registration) |
| Headplane admin UI | `https://headscale.home.philippthesurfer.com/admin` | LAN / VPN only |

## Deploy

```bash
ansible-playbook ansible/site.yml --tags headscale
```

## Notes

**HTTP/2 disabled on the Headscale router.** The `no-h2@file` TLS option is set in the Traefik label because some Tailscale clients have compatibility issues with HTTP/2 on the control plane endpoint.

**Adding devices:** Use Headplane at `/admin` to generate pre-auth keys, then run `tailscale up --login-server=https://vpn.philippthesurfer.com --authkey=<key>` on the device.

**STUN port 3478 (UDP)** must be forwarded from the FritzBox for NAT traversal to work reliably for remote peers.
