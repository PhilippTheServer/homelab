# Homelab

Deliberately over-engineered self-hosted infrastructure on a single mini PC. Everything is Infrastructure as Code — no manual steps after initial host setup.

For the full network diagram and architecture decisions, see [INFRASTRUCTURE.md](INFRASTRUCTURE.md).

---

# Setup

**Requirements**
- ansible
- ansible-galaxy (for community playbooks)
- Namecheap (domain)
- Cloudflare [API Token] (free account for tunel)
- ssh key to target system


To generate all passwords use:

```sh
# Strong random password (use for any "CHOOSE_A_PASSWORD" above)
openssl rand -base64 32

# Vaultwarden admin token
openssl rand -hex 32

# Harbor secret key (exactly 16 chars)
openssl rand -hex 8

# Traefik dashboard auth (run locally, needs apache2-utils)
htpasswd -nB philipp
```

Create a Cloudflare API Token. Paste into the vault file.



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
| 1 | **Keycloak** | SSO / OIDC identity provider — all services authenticate here | `auth.home.philippthesurfer.com` |
| 2 | **Traefik** | Reverse proxy, wildcard SSL (Let's Encrypt via Cloudflare DNS) | `traefik.home.philippthesurfer.com` |
| 2 | **Headscale** | Self-hosted Tailscale VPN control plane | `vpn.philippthesurfer.com` (public, via Cloudflare Tunnel) |
| 3 | **HashiCorp Vault** | Secrets management, dynamic credentials for automation | `vault.home.philippthesurfer.com` |
| 3 | **Vaultwarden** | Self-hosted Bitwarden-compatible password manager | `bw.home.philippthesurfer.com` |
| 4 | **Pi-hole** | DNS + DHCP, ad/tracker blocking, internal DNS resolution | `pihole.home.philippthesurfer.com` |
| 4 | **GitLab CE** | Git hosting + CI/CD pipelines | `gitlab.home.philippthesurfer.com` |
| 4 | **Harbor** | Docker image registry | `registry.home.philippthesurfer.com` |

All services sit behind Traefik with a wildcard Let's Encrypt cert (`*.home.philippthesurfer.com`). Each service stack is self-contained with its own Postgres (and Redis where needed) — stacks can be rebuilt or replaced independently.

---

## DNS Setup (Cloudflare Required)

Cloudflare is required — for two reasons:

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
  └── all secrets fetched at runtime from HashiCorp Vault (KV at secret/data/ansible)

After bootstrap:
  GitLab CI → re-deploys stacks on git push (Ansible only needed for host-level changes)
  CI/CD authenticates to Vault via AppRole → fetches secrets → runs playbook
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

Edit [ansible/inventory/hosts.yml](ansible/inventory/hosts.yml) to set target device.

### 2. Set non-secret variables

Edit [ansible/group_vars/all/vars.yml](ansible/group_vars/all/vars.yml) to change versions or other chore variables.


### 3. Populate secrets in HashiCorp Vault

All secrets live at `secret/data/ansible` in HashiCorp Vault. Ansible fetches them at runtime — nothing sensitive is stored in this repo.

First, install the required Ansible collection:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

Then store all secrets (run from your Mac after `vault login -method=oidc`):

```bash
vault kv put secret/ansible \
  cloudflare_api_token="..." \
  cloudflare_tunnel_token="..." \
  traefik_dashboard_auth="..." \
  pihole_webpassword="..." \
  keycloak_admin_user="admin" \
  keycloak_admin_password="..." \
  keycloak_db_password="..." \
  hcvault_root_token="..." \
  hcvault_oidc_secret="..." \
  vaultwarden_db_password="..." \
  vaultwarden_admin_token="..." \
  vaultwarden_oidc_secret="..." \
  headscale_db_password="..." \
  headscale_oidc_secret="..." \
  headplane_api_key="..." \
  headplane_cookie_secret="..." \
  gitlab_root_password="..." \
  gitlab_oidc_secret="..." \
  harbor_admin_password="..." \
  harbor_db_password="..." \
  harbor_oidc_secret="..."
```

<details>
<summary>Password generation reference</summary>

```bash
# Strong random password (use for any "..." above)
openssl rand -base64 32

# Vaultwarden admin token
openssl rand -hex 32

# Harbor secret key (exactly 16 chars)
openssl rand -hex 8

# Traefik dashboard auth (needs apache2-utils)
htpasswd -nB philipp

# Cloudflare API Token: dashboard → My Profile → API Tokens → Create Token
# Permission: Zone:DNS:Edit for philippthesurfer.com

# Cloudflare Tunnel token: Zero Trust → Networks → Tunnels → your tunnel → Configure

# OIDC secrets: filled in after Keycloak is running — copy from Keycloak client credentials
```

</details>

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
6. Add Pi-hole DNS record: `<name>.home.philippthesurfer.com` → Mini PC LAN IP

---

<details>
<summary>Repo Layout</summary>

```
homelab/
├── ansible/
│   ├── inventory/
│   │   └── hosts.yml              # Mini PC connection details
│   ├── group_vars/all/
│   │   ├── vars.yml               # non-secret config (domain, versions, IPs)
│   │   └── vault.yml              # hashi_vault lookups — secrets live in HCP Vault
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
├── INFRASTRUCTURE.md              # network diagram + architecture decisions
├── HASHICORPVAULT.md              # Vault CLI setup, SSH certs, secret management, Ansible integration
└── BOOTSTRAP.md                   # step-by-step first-deploy guide
```

</details>

---

## Troubleshooting

**Can't reach a service at `*.home.philippthesurfer.com`**
- Confirm your device is on the LAN or connected via Headscale VPN
- Check Pi-hole has a DNS record for the hostname pointing to Mini PC LAN IP
- Check Traefik dashboard (`traefik.home.philippthesurfer.com`) — verify the router is registered

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

---

## SSH Access via HashiCorp Vault

SSH uses Vault as a certificate authority — no static keys on servers, full audit trail.

**One-time setup on your Mac:**

Add to `~/.ssh/config`:

```
Host *.home.philippthesurfer.com
    User philipp
    IdentityFile ~/.ssh/vault
    CertificateFile ~/.ssh/vault-signed.pub
```

**Each session (cert valid for 8h):**

```bash
# 1. Login to Vault via Keycloak
vault login -method=oidc

# 2. Sign your SSH key
SIGNED=$(vault write -field=signed_key ssh/sign/user public_key="$(cat ~/.ssh/vault.pub)")
echo "$SIGNED" > ~/.ssh/vault-signed.pub

# 3. SSH in
ssh minipc.home.philippthesurfer.com
```

**CI/CD authentication (AppRole):**

```bash
# Get credentials once and store in CI/CD secret variables:
vault read -field=role_id auth/approle/role/ci-cd/role-id
vault write -field=secret_id -f auth/approle/role/ci-cd/secret-id

# In the pipeline, before running ansible:
export VAULT_TOKEN=$(vault write -field=token auth/approle/login \
  role_id=$VAULT_ROLE_ID \
  secret_id=$VAULT_SECRET_ID)
ansible-playbook ansible/site.yml
```
