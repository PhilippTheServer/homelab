# jellyfin

Self-hosted media server for streaming movies, TV shows, and music.

## Overview

Jellyfin streams locally stored media to any device via a browser or native app. It handles transcoding, subtitle extraction, chapter thumbnails, and user-specific watch progress. Configuration and metadata are stored in a named Docker volume; the media library is a read-only bind mount from `{{ homelab_base_dir }}/media` on the host. Jellyfin has no external database dependency — it uses its own internal SQLite store inside the config volume.

Authentication defaults to local accounts. SSO via Keycloak can be added post-deploy using the [Jellyfin SSO plugin](https://github.com/9p4/jellyfin-plugin-sso) — see Notes.

## Prerequisites

- `common` role applied (Docker, `proxy` network)
- `traefik` role running (TLS termination)

## Containers

| Container | Image | Purpose |
|---|---|---|
| `jellyfin` | `jellyfin/jellyfin` | Media server + transcoding |

Networks: joins `proxy` only (Traefik). No internal network needed — Jellyfin has no companion services.

## Configuration

Files rendered by Ansible to the host:

| File | Purpose |
|---|---|
| `/opt/homelab/services/jellyfin/docker-compose.yml` | Stack definition |

Media directory created by Ansible:

| Path | Purpose |
|---|---|
| `/opt/homelab/media` | Root media directory — mounted read-only into Jellyfin at `/media` |

Organise media under `/opt/homelab/media` using any layout Jellyfin supports (e.g. `movies/`, `tv/`, `music/`). Add each subdirectory as a library in the Jellyfin admin UI.

## Secrets

All secrets live in HashiCorp Vault at `secret/data/ansible`.

| Vault field | Variable | Purpose |
|---|---|---|
| `jellyfin_oidc_secret` | `vault_jellyfin_oidc_secret` | Keycloak OIDC client secret (SSO plugin, configured post-deploy via UI) |
| `jellyfin_api_key` | `vault_jellyfin_api_key` | API key for the Homepage dashboard widget |

Both are set to `PLACEHOLDER` on initial deploy and filled in after Jellyfin is running.

**API key** — generate in Jellyfin admin UI → Dashboard → API Keys → + , then store in Vault and re-run the homepage role:

```bash
vault kv patch secret/ansible jellyfin_api_key="<api-key>"
ansible-playbook ansible/site.yml --tags homepage
```

**OIDC secret** — after Keycloak is running, create the `jellyfin` OIDC client, copy the secret from Keycloak → Credentials tab, then patch Vault:

```bash
vault kv patch secret/ansible jellyfin_oidc_secret="<keycloak-client-secret>"
```

The OIDC secret is used when configuring the SSO plugin through the Jellyfin admin UI (see Notes). There is no compose-level env var for it.

## URL

`https://media.home.philippthesurfer.com`

## Deploy

```bash
ansible-playbook ansible/site.yml --tags jellyfin
```

## Notes

**Initial setup:** On first visit, the Jellyfin setup wizard runs. Create the admin account, add libraries pointing to subdirectories of `/media`, and let the initial scan complete.

**Media directory:** Drop media onto the host at `/opt/homelab/media/` (e.g. via `scp`, `rsync`, or Samba). The volume is read-only inside the container — Jellyfin writes all metadata and thumbnails to the `jellyfin_config` volume.

**Transcoding:** Software transcoding works out of the box. For hardware acceleration (Intel Quick Sync, VA-API), add the appropriate device passthrough to the compose file and re-run the role. Example for Intel iGPU:

```yaml
devices:
  - /dev/dri:/dev/dri
```

**SSO via Keycloak (optional):** Jellyfin supports OIDC through the community SSO plugin. Steps:

1. In Keycloak, create an OIDC client `jellyfin` in the `homelab` realm. Set the redirect URI to `https://media.home.philippthesurfer.com/sso/OID/r/keycloak`.
2. In Jellyfin admin UI → Plugins → Catalog → install **SSO-Auth**.
3. Restart the Jellyfin container: `docker restart jellyfin`
4. In Jellyfin admin UI → Plugins → SSO-Auth → configure the Keycloak provider using the client ID and secret from step 1.

**Clients:** Use any Jellyfin client — the web UI, Jellyfin for Android/iOS, Infuse, Swiftfin, or Kodi with the Jellyfin add-on. Point them at `https://media.home.philippthesurfer.com`.
