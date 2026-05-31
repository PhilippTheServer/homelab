# keycloak

SSO / OIDC identity provider — all services authenticate here.

## Overview

Keycloak is the central authentication hub for the homelab. Every service that supports OIDC (Vaultwarden, GitLab, Harbor, HashiCorp Vault, Headscale, Homepage, Paperless-ngx) is configured as a client in the `homelab` realm. A realm export JSON is rendered by Ansible and imported automatically on first container startup via the `--import-realm` flag, so OIDC clients and realm settings are declared as code. Keycloak's database is an isolated Postgres container on an internal-only network.

## Prerequisites

- `common` role applied (Docker, `proxy` network)
- `traefik` role running (TLS termination)
- Pi-hole DNS record `auth.philippthesurfer.com` → `192.168.178.20` (added automatically by the `pihole` role)

## Containers

| Container | Image | Purpose |
|---|---|---|
| `keycloak` | `quay.io/keycloak/keycloak:26.6` | Identity provider |
| `keycloak-db` | `postgres:18-alpine` | Keycloak database |

Networks: `keycloak` joins `proxy` (Traefik-visible) and `keycloak_internal` (database-only, not routable).

## Configuration

Files rendered by Ansible to the host:

| File | Purpose |
|---|---|
| `/opt/homelab/services/keycloak/docker-compose.yml` | Stack definition |
| `/opt/homelab/services/keycloak/import/realm-homelab.json` | Realm definition imported on first boot |

## Secrets

All secrets live in HashiCorp Vault at `secret/data/ansible`.

| Vault field | Variable | Purpose |
|---|---|---|
| `keycloak_db_password` | `vault_keycloak_db_password` | Postgres password for the `keycloak` database |
| `keycloak_admin_user` | `vault_keycloak_admin_user` | Initial admin username |
| `keycloak_admin_password` | `vault_keycloak_admin_password` | Initial admin password |

## URL

`https://auth.philippthesurfer.com`

Keycloak is the only service that sits at the apex domain (not under `home.`) because OIDC redirect URIs must resolve from any device, including those not on the LAN or VPN.

## Deploy

```bash
ansible-playbook ansible/site.yml --tags keycloak
```

## Notes

**Realm import is first-boot only.** The `--import-realm` flag skips import if the realm already exists, so re-deploying is safe and will not overwrite manual changes made in the Keycloak UI. To fully re-import, delete the `keycloak_db` volume and redeploy.

**OIDC client secrets** are generated in the Keycloak admin UI after first boot, then stored in HashiCorp Vault (`vault kv patch secret/ansible <service>_oidc_secret="..."`). Ansible reads them back on subsequent runs to configure each service's OIDC integration.

**Setup priority:** Keycloak should be deployed first among all services because every other OIDC-enabled service depends on it being reachable to complete its own configuration.
