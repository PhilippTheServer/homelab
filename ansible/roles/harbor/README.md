# harbor

Docker container image registry.

## Overview

Harbor is an enterprise-grade container registry with role-based access control, vulnerability scanning, and image signing. Unlike other roles that manage a simple `docker-compose.yml`, this role uses Harbor's official online installer — it downloads a versioned installer tarball, runs the `prepare` script (which generates Harbor's own internal compose files from `harbor.yml`), and then starts the stack using an override compose file to integrate Harbor into the existing Traefik `proxy` network. After startup, the role configures OIDC authentication via the Harbor API so Keycloak handles all logins.

## Prerequisites

- `common` role applied (Docker, `proxy` network)
- `traefik` role running (TLS termination)
- `keycloak` role running (OIDC SSO — the OIDC configuration step polls the Harbor API)

## Containers

Harbor's installer generates its own compose file. Key containers:

| Container | Purpose |
|---|---|
| `harbor-core` | Main Harbor API and web UI |
| `harbor-db` | Built-in Postgres database |
| `harbor-jobservice` | Async job processing (replication, GC) |
| `nginx` | Internal Harbor proxy |
| `registry` | Docker distribution registry backend |
| `redis` | Cache and job queue |

## Configuration

Files rendered by Ansible to the host:

| File | Purpose |
|---|---|
| `/opt/homelab/harbor/harbor.yml` | Primary Harbor configuration (hostname, HTTP port, storage paths, database password) |
| `/opt/homelab/harbor/docker-compose.override.yml` | Attaches Harbor containers to the `proxy` Docker network and adds Traefik labels |

Harbor listens internally on port `18080` (plain HTTP). TLS is fully handled by Traefik.

## Secrets

All secrets live in HashiCorp Vault at `secret/data/ansible`.

| Vault field | Variable | Purpose |
|---|---|---|
| `harbor_admin_password` | `vault_harbor_admin_password` | Initial admin password |
| `harbor_db_password` | `vault_harbor_db_password` | Built-in Postgres password |
| `harbor_oidc_secret` | `vault_harbor_oidc_secret` | Keycloak OIDC client secret |

## URL

`https://harbor.home.philippthesurfer.com`

## Deploy

```bash
ansible-playbook ansible/site.yml --tags harbor
```

## Notes

**Installer-based setup:** The `prepare` step only runs once (guarded by `creates:`). To force a full re-initialization, delete `/opt/homelab/harbor/common/config/` and re-run the role.

**OIDC is configured idempotently via the API.** The role reads the current `auth_mode` from the Harbor API and only applies the OIDC configuration if it is not already set to `oidc_auth`. To reconfigure OIDC without a full redeploy, set `auth_mode` back to `db_auth` in the Harbor admin UI and re-run the role.

**Pushing images** requires logging in with the OIDC-issued CLI secret (not the Keycloak password directly). In Harbor UI go to *User Profile → CLI secret*, copy it, then:

```bash
docker login harbor.home.philippthesurfer.com
# username: <keycloak username>
# password: <CLI secret from Harbor UI>
```
