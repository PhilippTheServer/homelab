# monitoring

Metrics and log aggregation stack: Grafana + Prometheus + Loki.

## Overview

This role deploys a full observability stack as a single Docker Compose stack. Grafana is the unified UI for both metrics (Prometheus) and logs (Loki). All access to Grafana is via Keycloak SSO — the login form is disabled and the browser redirects directly to Keycloak on every visit. Prometheus and Loki are internal-only containers; they are never exposed through Traefik.

Traefik's built-in Prometheus endpoint (`:8082/metrics`) is scraped automatically. Prometheus also runs on the `proxy` Docker network to reach the Traefik container directly by name.

## Prerequisites

- `common` role applied (Docker, `proxy` network)
- `traefik` role running (TLS termination + metrics endpoint enabled)
- `keycloak` role running (OIDC SSO)

## Containers

| Container | Purpose | Port (internal) |
|---|---|---|
| `grafana` | Dashboard UI — Prometheus + Loki, SSO via Keycloak | 3000 |
| `prometheus` | Metrics scraper + TSDB (30-day retention) | 9090 |
| `loki` | Log aggregation — receives logs from Promtail | 3100 |
| `promtail` | Docker log shipper — tails all container logs → Loki | 9080 |
| `node-exporter` | Host metrics (CPU, memory, disk, network) | 9100 |
| `cadvisor` | Per-container resource metrics | 8080 |

## Configuration

| File | Purpose |
|---|---|
| `/opt/homelab/services/monitoring/docker-compose.yml` | Stack definition |
| `/opt/homelab/services/monitoring/prometheus/prometheus.yml` | Scrape targets |
| `/opt/homelab/services/monitoring/loki/loki.yml` | Loki single-binary config (filesystem storage) |
| `/opt/homelab/services/monitoring/promtail/promtail.yml` | Docker log discovery and shipping config |
| `/opt/homelab/services/monitoring/grafana/provisioning/datasources/datasources.yml` | Auto-configures Prometheus + Loki datasources |
| `/opt/homelab/services/monitoring/grafana/provisioning/dashboards/dashboards.yml` | Dashboard file provider pointing to `/etc/grafana/dashboards` |
| `/opt/homelab/services/monitoring/grafana/dashboards/*.json` | Pre-provisioned dashboards (Node Exporter Full, cAdvisor) — downloaded by Ansible at deploy time |

Prometheus retains 30 days of metrics. Loki uses on-disk TSDB storage under a named Docker volume.

## Secrets

All secrets live in HashiCorp Vault at `secret/data/ansible`.

| Vault field | Variable | Purpose |
|---|---|---|
| `grafana_admin_password` | `vault_grafana_admin_password` | Bootstrap admin password (emergency access via API only — login form is disabled) |
| `grafana_oidc_secret` | `vault_grafana_oidc_secret` | Keycloak OIDC client secret |

## URL

`https://monitoring.home.philippthesurfer.com`

## Deploy

```bash
# Add secrets first (one-time)
vault kv patch secret/ansible \
  grafana_admin_password="$(openssl rand -base64 32)" \
  grafana_oidc_secret="PLACEHOLDER"

# Wire Keycloak client, copy the secret, then patch:
vault kv patch secret/ansible grafana_oidc_secret="<secret-from-keycloak>"

ansible-playbook ansible/site.yml --tags monitoring
```

## Notes

**SSO is enforced.** `GF_AUTH_DISABLE_LOGIN_FORM=true` and `GF_AUTH_GENERIC_OAUTH_AUTO_LOGIN=true` mean the browser goes directly to Keycloak. The admin password in Vault is only useful for API access (`Authorization: Basic`). To temporarily re-enable the login form for emergency recovery, set `GF_AUTH_DISABLE_LOGIN_FORM=false` and redeploy.

**Role mapping.** Keycloak realm role `admin` → Grafana `Admin`. All other authenticated users get `Viewer`. To promote a user to Editor, assign them the `editor` Keycloak role and update the `GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH` expression, then redeploy.

**Traefik metrics.** The role expects `--metrics.prometheus=true` and `--entrypoints.metrics.address=:8082` in the Traefik command. These flags are included in the Traefik role's `docker-compose.yml`. Re-run `--tags traefik` if Traefik was deployed before this role.

**Dashboards.** Two community dashboards are provisioned at deploy time by downloading their JSON from grafana.com: Node Exporter Full (ID 1860) and cAdvisor (ID 14282). Additional dashboards can be added by placing JSON files in `/opt/homelab/services/monitoring/grafana/dashboards/` — Grafana picks them up within 30 seconds without a restart.

**Loki log retention.** There is no default retention policy. To add one, set `retention_period` under `limits_config` in `loki.yml.j2` and re-run the role.
