# Homelab

Deliberately over-engineered self-hosted infrastructure on a single mini PC. Everything is Infrastructure as Code — no manual steps after initial host setup.

For the full network diagram and architecture decisions, see [INFRASTRUCTURE.md](INFRASTRUCTURE.md).

---

## Access Model

**There is no public IP and no open ports on the Fritz Box.** Services are accessible in two ways only:

- **On the local network** — `*.home.philippthesurfer.com` resolves via Pi-hole to the Mini PC's LAN IP
- **Remotely via VPN** — Headscale (self-hosted Tailscale) provides a private mesh network

The only exception is Headscale's own control plane, which must be reachable from the internet for VPN device registration. This is handled via a **Cloudflare Tunnel** — a single outbound `cloudflared` container creates a tunnel to Cloudflare's edge, routing `vpn.philippthesurfer.com` → Headscale. No ports are opened. Nothing else is tunneled.

**Keycloak, Bitwarden, and all other services are never reachable from the public internet.**

---

## Stack

Services are listed in setup priority order — the order in which they should be configured.

| Priority | Service | Role | URL |
|---|---|---|---|
| 1 | **Keycloak** | SSO / OIDC identity provider — all services authenticate here | `auth.home.<domain>` |
| 2 | **Traefik** | Reverse proxy, wildcard SSL (Let's Encrypt via Cloudflare DNS) | `traefik.home.<domain>` |
| 2 | **Headscale** | Self-hosted Tailscale VPN control plane | `vpn.<domain>` (public, via Cloudflare Tunnel) |
| 3 | **HashiCorp Vault** | Secrets management, dynamic credentials for automation | `vault.home.<domain>` |
| 3 | **Vaultwarden** | Self-hosted Bitwarden-compatible password manager | `bw.home.<domain>` |
| 4 | **Pi-hole** | DNS + DHCP, ad/tracker blocking, internal DNS resolution | `pihole.home.<domain>` |
| 4 | **GitLab CE** | Git hosting + CI/CD pipelines | `gitlab.home.<domain>` |
| 4 | **Harbor** | Docker image registry | `registry.home.<domain>` |

All services sit behind Traefik with a wildcard Let's Encrypt cert (`*.home.<domain>`). Each service stack is self-contained with its own Postgres (and Redis where needed) — stacks can be rebuilt or replaced independently.

---

## DNS Setup (Cloudflare Required)

Cloudflare is required — not just recommended — for two reasons:

1. **Cloudflare Tunnel** is how Headscale is reachable from the internet without a public IP
2. **Cloudflare DNS challenge** is how Traefik gets Let's Encrypt certs without exposing any port

**Steps:**
1. Keep your domain registered at Namecheap
2. Create a free Cloudflare account and add your domain
3. Namecheap → Domain settings → change nameservers to Cloudflare's nameservers
4. In Cloudflare: create an API token with `Zone:DNS:Edit` permission (used by Traefik)
5. In Cloudflare: create a Tunnel for Headscale (Cloudflare Zero Trust → Tunnels → Create tunnel)
   - Route: `vpn.philippthesurfer.com` → `http://headscale:8080` (internal container hostname)
   - Copy the tunnel token — goes into Ansible Vault

Internal service DNS (`*.home.philippthesurfer.com`) is handled by Pi-hole pointing to the Mini PC LAN IP. These records never leave your network.

---

## How It Works

```
Ansible (local machine)
  └── renders Jinja2 templates → Docker Compose files on the Mini PC
  └── runs docker compose up -d per stack
  └── all secrets come from Ansible Vault (encrypted in this repo)

After bootstrap:
  GitLab CI → re-deploys stacks on git push (Ansible only needed for host-level changes)
```

---

## Prerequisites

On your **local machine** (where you run Ansible):

- Ansible `>= 2.15` — `pip install ansible`
- SSH key added to the Mini PC's `~/.ssh/authorized_keys`
- Git

On the **Mini PC** (one-time manual steps before first Ansible run):

```bash
# Install SSH server if not present
apt install openssh-server -y

# Create deploy user
useradd -m -s /bin/bash deploy
usermod -aG sudo deploy

# Add your SSH public key
mkdir -p /home/deploy/.ssh
echo "YOUR_PUBLIC_KEY" >> /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh && chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

On **Cloudflare** (before running Ansible):

1. Domain added to Cloudflare with nameservers updated at Namecheap
2. API token created: `Zone:DNS:Edit` for your domain
3. Cloudflare Tunnel created for Headscale — tunnel token copied
4. DNS record added: `vpn.philippthesurfer.com` → tunnel (Cloudflare sets this automatically when creating the tunnel)

---

## Setup

### 1. Configure inventory

Edit [ansible/inventory/hosts.yml](ansible/inventory/hosts.yml):

```yaml
all:
  hosts:
    minipc:
      ansible_host: 192.168.178.x   # Mini PC LAN IP
      ansible_user: deploy
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

### 2. Set non-secret variables

Edit [ansible/group_vars/all/vars.yml](ansible/group_vars/all/vars.yml):

```yaml
domain: "yourdomain.com"
internal_subdomain: "home"          # services at *.home.yourdomain.com
minipc_ip: "192.168.178.x"
traefik_acme_email: "you@email.com"
letsencrypt_staging: false          # set true during initial setup to avoid rate limits

# Pin versions — update deliberately
traefik_version: "v3.x"
keycloak_version: "25.x.x"
headscale_version: "0.x.x"
vault_version: "1.x.x"
vaultwarden_version: "1.x.x"
gitlab_version: "17.x.x-ce.0"
harbor_version: "v2.x.x"
pihole_version: "latest"
```

### 3. Create the Ansible Vault

```bash
ansible-vault create ansible/group_vars/all/vault.yml
```

Populate with (use `ansible-vault edit` to update later):

```yaml
# Cloudflare (Traefik DNS challenge + Tunnel)
vault_cloudflare_api_token: "your_cloudflare_api_token"
vault_cloudflare_tunnel_token: "your_cloudflare_tunnel_token"  # for cloudflared container

# Traefik dashboard
vault_traefik_dashboard_auth: "admin:$2y$..."  # generate: htpasswd -nB admin

# Keycloak admin
vault_keycloak_admin_user: "admin"
vault_keycloak_admin_password: "changeme"
vault_keycloak_db_password: "changeme"

# HashiCorp Vault
vault_hcvault_db_password: "changeme"
# Note: Vault unseal keys and root token are generated on first init — store in Bitwarden

# Vaultwarden
vault_vaultwarden_db_password: "changeme"
vault_vaultwarden_admin_token: "changeme"  # generate: openssl rand -hex 32

# GitLab
vault_gitlab_root_password: "changeme"
vault_gitlab_db_password: "changeme"

# Harbor
vault_harbor_admin_password: "changeme"
vault_harbor_db_password: "changeme"
vault_harbor_secret_key: "changeme-16chars"  # exactly 16 characters

# Headscale
vault_headscale_db_password: "changeme"

# Keycloak OIDC client secrets (fill in after Keycloak is running)
vault_gitlab_oidc_secret: ""
vault_harbor_oidc_secret: ""
vault_headscale_oidc_secret: ""
vault_vaultwarden_oidc_secret: ""
```

Store your vault password in a password manager. Never commit it.

---

## Bootstrap

### Deployment order

Infrastructure must be bootstrapped in dependency order. Priority reflects architectural importance; deployment order reflects technical dependencies.

```
[infrastructure]  common     → Docker, UFW, deploy user (prerequisite for everything)
[infrastructure]  traefik    → SSL and routing (prerequisite for all HTTPS services)

[priority 1]      keycloak   → SSO — must exist before any service is wired to it

[priority 2]      headscale  → VPN — wire Keycloak OIDC, then Cloudflare Tunnel goes live

[priority 3]      vault      → HashiCorp Vault — secrets automation
[priority 3]      vaultwarden → Password manager

[priority 4]      pihole     → Internal DNS (or migrate existing Pi-hole to IaC)
[priority 4]      gitlab     → Push this repo here; CI takes over re-deploys
[priority 4]      harbor     → Registry
```

### Commands

```bash
# 0. Verify connectivity
ansible minipc -m ping -i ansible/inventory/hosts.yml --ask-vault-pass

# 1. Provision host
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags common --ask-vault-pass

# 2. Traefik (SSL prerequisite)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags traefik --ask-vault-pass

# 3. Keycloak
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags keycloak --ask-vault-pass

# 4. Headscale (wire Keycloak OIDC first — see post-bootstrap below)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags headscale --ask-vault-pass

# 5. HashiCorp Vault
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags hcvault --ask-vault-pass

# 6. Vaultwarden
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags vaultwarden --ask-vault-pass

# 7. Pi-hole (migrate existing or fresh deploy)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags pihole --ask-vault-pass

# 8. GitLab + Harbor
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags gitlab,harbor --ask-vault-pass
```

### Post-bootstrap: wire SSO

After Keycloak is running, create OIDC clients for each service before deploying them:

1. Log into `auth.home.<domain>` → create realm `homelab`
2. Create OIDC clients: `headscale`, `gitlab`, `harbor`, `vaultwarden`
3. Add each client secret to Vault:
   ```bash
   ansible-vault edit ansible/group_vars/all/vault.yml
   # fill in vault_*_oidc_secret fields
   ```
4. Re-run the affected roles to apply SSO config

### Post-bootstrap: initialize HashiCorp Vault

HashiCorp Vault requires a one-time init after first deploy:

```bash
ssh deploy@192.168.178.x
docker exec -it vault vault operator init

# Save the 5 unseal keys and root token in Vaultwarden immediately
# Vault needs 3 of 5 keys to unseal after every restart
docker exec -it vault vault operator unseal  # run 3 times with different keys
```

Consider configuring auto-unseal later (e.g. using a cloud KMS or a dedicated key) to avoid manual unsealing after reboots.

### Post-bootstrap: migrate this repo to GitLab

```bash
git remote add homelab https://gitlab.home.<domain>/<user>/homelab.git
git push homelab main
```

From this point, GitLab CI handles re-deploys on push.

---

## Day-to-Day

### Update a service version

1. Edit the version in `ansible/group_vars/all/vars.yml`
2. Push to GitLab → CI re-deploys the affected stack
3. Or manually: `ansible-playbook ansible/site.yml --tags <service> --ask-vault-pass`

### Restart a stack

```bash
ssh deploy@192.168.178.x
docker compose -f /opt/homelab/services/<service>/docker-compose.yml restart
```

### View logs

```bash
ssh deploy@192.168.178.x
docker compose -f /opt/homelab/services/<service>/docker-compose.yml logs -f
```

### Rotate a secret

1. `ansible-vault edit ansible/group_vars/all/vault.yml` — update the value
2. Re-run the role: `ansible-playbook ansible/site.yml --tags <service> --ask-vault-pass`

---

## Adding a New Service

1. Create `services/<name>/docker-compose.yml.j2` (Jinja2 template)
2. Create `ansible/roles/<name>/tasks/main.yml`:
   - Creates target directory on Mini PC
   - Renders template via `template:` module
   - Runs `docker compose up -d`
3. Add secrets to `vault.yml`, variables to `vars.yml`
4. Add role to `ansible/site.yml` with matching tag
5. Create OIDC client in Keycloak if the service supports SSO
6. Add Pi-hole DNS record: `<name>.home.<domain>` → Mini PC LAN IP

---

## Repo Layout

```
homelab/
├── ansible/
│   ├── inventory/
│   │   └── hosts.yml              # Mini PC connection details
│   ├── group_vars/all/
│   │   ├── vars.yml               # non-secret config (domain, versions, IPs)
│   │   └── vault.yml              # ansible-vault encrypted secrets
│   ├── roles/
│   │   ├── common/                # Docker, UFW, deploy user
│   │   ├── traefik/               # reverse proxy + SSL + cloudflared tunnel
│   │   ├── keycloak/              # SSO
│   │   ├── headscale/             # VPN
│   │   ├── hcvault/               # HashiCorp Vault
│   │   ├── vaultwarden/           # Bitwarden-compatible password manager
│   │   ├── pihole/                # DNS + DHCP
│   │   ├── gitlab/                # Git + CI/CD
│   │   └── harbor/                # Docker registry
│   └── site.yml                   # master playbook
├── services/
│   ├── traefik/                   # includes cloudflared sidecar
│   ├── keycloak/
│   ├── headscale/
│   ├── hcvault/
│   ├── vaultwarden/
│   ├── pihole/
│   ├── gitlab/
│   └── harbor/
├── README.md
└── INFRASTRUCTURE.md              # network diagram + architecture decisions
```

---

## Troubleshooting

**Can't reach a service at `*.home.<domain>`**
- Confirm your device is on the LAN or connected via Headscale VPN
- Check Pi-hole has a DNS record for the hostname pointing to Mini PC LAN IP
- Check Traefik dashboard (`traefik.home.<domain>`) — verify the router is registered

**Let's Encrypt cert not issuing**
- Check Traefik logs: `docker compose -f /opt/homelab/services/traefik/docker-compose.yml logs`
- Verify `vault_cloudflare_api_token` in Ansible Vault is correct and has `Zone:DNS:Edit` permission
- Set `letsencrypt_staging: true` and re-deploy to test without hitting rate limits

**Headscale devices can't connect remotely**
- Check Cloudflare Tunnel status: Cloudflare Zero Trust dashboard → Tunnels
- Check `cloudflared` container logs in the traefik stack
- Verify `vpn.philippthesurfer.com` resolves to Cloudflare (not your LAN IP)

**Keycloak OIDC login failing**
- Verify the redirect URI in the Keycloak client matches the service URL exactly
- Check client secret in Vault matches Keycloak
- Re-run the affected role after any Vault change

**HashiCorp Vault sealed after reboot**
- Vault seals itself on restart by design — unseal manually with 3 of 5 keys:
  ```bash
  docker exec -it vault vault operator unseal
  ```
- Long-term: configure auto-unseal with a cloud KMS or dedicated unsealing key
