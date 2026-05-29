# Homelab

Deliberately over-engineered self-hosted infrastructure on a single mini PC. Everything is Infrastructure as Code — no manual steps after initial host setup.

For the full network diagram and architecture decisions, see [INFRASTRUCTURE.md](INFRASTRUCTURE.md).

---

# Setup

**Requirements**
- ansible
- ansible-galaxy (for community playbooks)
- Namecheap domain with DDNS enabled and API access
- SSH key to target system

To generate all passwords use:

```sh
# Strong random password (use for any "CHOOSE_A_PASSWORD" below)
openssl rand -base64 32

# Vaultwarden admin token
openssl rand -hex 32

# Harbor secret key (exactly 16 chars)
openssl rand -hex 8

# Traefik dashboard auth (run locally, needs apache2-utils)
htpasswd -nB philipp
```

---

## Access Model

**Remote access uses open ports — TCP 443, TCP 80, and UDP 3478 are forwarded on the FritzBox to the Mini PC.** Services are accessible in two ways:

- **On the local network** — `*.home.philippthesurfer.com` resolves via Pi-hole to the Mini PC's LAN IP
- **Remotely via VPN** — Headscale (self-hosted Tailscale) provides a private mesh network

Headscale's control plane is reachable from the internet via `vpn.philippthesurfer.com`, which is a public DNS A record kept current by ddclient (Namecheap DDNS). Traefik terminates TLS directly — no tunnel or third-party proxy sits in front of it.

**Keycloak, Bitwarden, and all other services are never reachable from the public internet.**

---

## Stack

Services are listed in setup priority order — the order in which they should be configured.

| Priority | Service | Role | URL |
|---|---|---|---|
| 1 | **Keycloak** | SSO / OIDC identity provider — all services authenticate here | `auth.home.philippthesurfer.com` |
| 2 | **Traefik** | Reverse proxy, wildcard SSL (Let's Encrypt via Namecheap DNS) | `traefik.home.philippthesurfer.com` |
| 2 | **Headscale** | Self-hosted Tailscale VPN control plane | `vpn.philippthesurfer.com` (public, direct via DDNS) |
| 3 | **HashiCorp Vault** | Secrets management, dynamic credentials for automation | `vault.home.philippthesurfer.com` |
| 3 | **Vaultwarden** | Self-hosted Bitwarden-compatible password manager | `bw.home.philippthesurfer.com` |
| 4 | **Pi-hole** | DNS + DHCP, ad/tracker blocking, internal DNS resolution | `pihole.home.philippthesurfer.com` |
| 4 | **GitLab CE** | Git hosting + CI/CD pipelines | `gitlab.home.philippthesurfer.com` |
| 4 | **Harbor** | Docker image registry | `registry.home.philippthesurfer.com` |

All services sit behind Traefik with a wildcard Let's Encrypt cert (`*.home.philippthesurfer.com`). Each service stack is self-contained with its own Postgres (and Redis where needed) — stacks can be rebuilt or replaced independently.

---

## DNS & External Access

The domain is registered at Namecheap. Two Namecheap features are used:

1. **DDNS** — ddclient keeps the A record for `vpn.philippthesurfer.com` pointing at the current home IP, updated every 5 minutes
2. **DNS challenge** — Traefik uses the Namecheap API to issue wildcard Let's Encrypt certs (`*.home.philippthesurfer.com`) without requiring those internal domains to be publicly reachable

See [DDNS.md](DDNS.md) for full setup and troubleshooting details.

**Steps to configure Namecheap before running Ansible:**
1. Domain List → Manage → Advanced DNS → enable **Dynamic DNS** (note the DDNS password)
2. Profile → Tools → API Access → enable API and whitelist your home IP (`curl -4 -s https://icanhazip.com`)
3. Advanced DNS → add A record: `vpn` → current home IP (ddclient will keep it updated)

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

On **Namecheap** (before running Ansible):

1. DDNS enabled for the domain — DDNS password noted
2. API access enabled — home IP whitelisted
3. A record `vpn.philippthesurfer.com` created

On **FritzBox** (before deploying Traefik):

Navigate to **http://fritz.box → Internet → Permit Access → Port Sharing**, target `192.168.178.20`:

| Protocol | Port | Purpose |
|----------|------|---------|
| TCP | 443 | Traefik HTTPS / Headscale |
| TCP | 80 | Let's Encrypt HTTP fallback |
| UDP | 3478 | STUN for Tailscale NAT traversal |

---

## Setup

### 1. Configure inventory

Edit [ansible/inventory/hosts.yml](ansible/inventory/hosts.yml) to set target device.

### 2. Set non-secret variables

Edit [ansible/group_vars/all/vars.yml](ansible/group_vars/all/vars.yml) to change versions or other config variables.

### 3. Populate secrets in HashiCorp Vault

All secrets live at `secret/data/ansible` in HashiCorp Vault. Ansible fetches them at runtime — nothing sensitive is stored in this repo.

First, install the required Ansible collection:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

Then store all secrets (run from your Mac after `vault login -method=oidc`):

```bash
vault kv put secret/ansible \
  namecheap_api_user="..." \
  namecheap_api_key="..." \
  namecheap_ddns_password="..." \
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

# Namecheap API key: Profile → Tools → API Access
# Namecheap DDNS password: Domain List → Manage → Advanced DNS → Dynamic DNS
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
3. Add secret to HCVault (`vault kv patch secret/ansible <field>=<value>`) and its `vault_*` mapping to `vars.yml`
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
│   │   └── vars.yml               # config (domain, versions, IPs) + HCVault lookups
│   ├── roles/
│   │   ├── common/                # Docker, UFW, deploy user
│   │   ├── traefik/               # reverse proxy + SSL + ddclient DDNS
│   │   ├── keycloak/              # SSO
│   │   ├── headscale/             # VPN
│   │   ├── hcvault/               # HashiCorp Vault
│   │   ├── vaultwarden/           # Bitwarden-compatible password manager
│   │   ├── pihole/                # DNS + DHCP
│   │   ├── gitlab/                # Git + CI/CD
│   │   ├── harbor/                # Docker registry
│   │   └── homepage/              # Dashboard
│   └── site.yml                   # master playbook
├── services/
│   ├── traefik/                   # includes ddclient DDNS sidecar
│   ├── keycloak/
│   ├── headscale/
│   ├── hcvault/
│   ├── vaultwarden/
│   ├── pihole/
│   ├── gitlab/
│   ├── harbor/
│   └── homepage/
├── README.md
├── INFRASTRUCTURE.md              # network diagram + architecture decisions
├── DDNS.md                        # DDNS setup, Namecheap config, cert issuance
├── HASHICORPVAULT.md              # Vault CLI setup, SSH certs, secret management, Ansible integration
└── BOOTSTRAP.md                   # step-by-step first-deploy guide
```

</details>

---

## Role Documentation

Each Ansible role has its own README covering overview, containers, configuration files, required secrets, URL, and operational notes.

```
ansible/roles/
├── common/README.md       — Docker, UFW firewall, deploy user, proxy network
├── traefik/README.md      — Reverse proxy, wildcard TLS, ddclient DDNS
├── keycloak/README.md     — SSO / OIDC identity provider
├── headscale/README.md    — Self-hosted Tailscale VPN control plane + Headplane UI
├── hcvault/README.md      — Secrets management, AppRole CI/CD auth, SSH CA
├── vaultwarden/README.md  — Bitwarden-compatible password manager
├── pihole/README.md       — DNS + DHCP + ad blocking + internal name resolution
├── gitlab/README.md       — Git hosting + CI/CD pipelines + GitLab Runner
├── harbor/README.md       — Docker container image registry
└── homepage/README.md     — Homelab dashboard with OIDC-protected access
```

| Role | README |
|---|---|
| common | [ansible/roles/common/README.md](ansible/roles/common/README.md) |
| traefik | [ansible/roles/traefik/README.md](ansible/roles/traefik/README.md) |
| keycloak | [ansible/roles/keycloak/README.md](ansible/roles/keycloak/README.md) |
| headscale | [ansible/roles/headscale/README.md](ansible/roles/headscale/README.md) |
| hcvault | [ansible/roles/hcvault/README.md](ansible/roles/hcvault/README.md) |
| vaultwarden | [ansible/roles/vaultwarden/README.md](ansible/roles/vaultwarden/README.md) |
| pihole | [ansible/roles/pihole/README.md](ansible/roles/pihole/README.md) |
| gitlab | [ansible/roles/gitlab/README.md](ansible/roles/gitlab/README.md) |
| harbor | [ansible/roles/harbor/README.md](ansible/roles/harbor/README.md) |
| homepage | [ansible/roles/homepage/README.md](ansible/roles/homepage/README.md) |

---

## Troubleshooting

**Can't reach a service at `*.home.philippthesurfer.com`**
- Confirm your device is on the LAN or connected via Headscale VPN
- Check Pi-hole has a DNS record for the hostname pointing to Mini PC LAN IP
- Check Traefik dashboard (`traefik.home.philippthesurfer.com`) — verify the router is registered

**Let's Encrypt cert not issuing**
- Check Traefik logs: `docker compose -f /opt/homelab/services/traefik/docker-compose.yml logs traefik`
- Verify `vault_namecheap_api_key` is correct and the home IP is whitelisted in Namecheap API settings
- Set `letsencrypt_staging: true` and re-deploy to test without hitting rate limits

**Headscale devices can't connect remotely**
- Check ddclient logs: `docker logs ddclient` — confirm A record is current
- Verify `vpn.philippthesurfer.com` resolves to your home IP: `dig vpn.philippthesurfer.com +short`
- Check FritzBox port forwarding is set up: TCP 443 → 192.168.178.20
- See [DDNS.md](DDNS.md) for the full diagnostic checklist

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
