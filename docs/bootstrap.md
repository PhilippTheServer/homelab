# Bootstrap

Step-by-step guide for the first deploy of the homelab. After bootstrap, GitLab CI handles re-deploys.

---

## Prerequisites (control machine)

```bash
# macOS: prevent "A worker was found in a dead state" fork-safety crash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES  # add to ~/.zshrc to make permanent

# Install hvac into Ansible's Python — required for community.hashi_vault lookups
# Pin to 1.x: hvac 2.x changed response types, breaking community.hashi_vault 7.x lookups
/opt/homebrew/Cellar/ansible/*/libexec/bin/python -m pip install "hvac"

# Install Ansible collections
ansible-galaxy collection install -r ansible/requirements.yml
```

---

## Deployment Order

Infrastructure must be bootstrapped in dependency order.

```
[infrastructure]  common      → Docker, UFW, deploy user (prerequisite for everything)
[infrastructure]  traefik     → SSL and routing (prerequisite for all HTTPS services)

[priority 1]      keycloak    → SSO — must exist before any service is wired to it

[priority 2]      headscale   → VPN — wire Keycloak OIDC, then port forward goes live

[priority 3]      hcvault     → HashiCorp Vault — secrets automation
[priority 3]      vaultwarden → Password manager

[priority 4]      pihole      → Internal DNS (or migrate existing Pi-hole to IaC)
[priority 4]      gitlab      → Push this repo here; CI takes over re-deploys
[priority 4]      harbor      → Registry
[priority 4]      homepage    → Dashboard
[priority 4]      paperless   → Document management
[priority 4]      jellyfin    → Media server
[priority 4]      monitoring  → Metrics + log aggregation
```

<details>
<summary>Ansible commands for each step</summary>

```bash
# Pre-requisite: authenticate to HashiCorp Vault (sets ~/.vault-token, valid 8h)
export VAULT_ADDR=https://vault.home.philippthesurfer.com
vault login -method=oidc

# Sign your SSH key — Ansible connects to the hosts with this Vault-signed cert
vault write -field=signed_key ssh/sign/user \
  public_key="$(cat ~/.ssh/hcvault.pub)" > ~/.ssh/hcvault-cert.pub

# 0. Verify connectivity
ansible minipc -m ping -i ansible/inventory/hosts.yml

# 1. Provision host
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags common

# 2. Traefik (SSL + DDNS prerequisite)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags traefik

# 3. Keycloak
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags keycloak

# 4. Headscale (wire Keycloak OIDC first — see post-deploy section below)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags headscale

# 5. HashiCorp Vault
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags hcvault

# 6. Vaultwarden
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags vaultwarden

# 7. Pi-hole
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags pihole

# 8. GitLab + Harbor + Homepage + Paperless + Jellyfin + Monitoring
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags gitlab,harbor,homepage,paperless,jellyfin,monitoring
```

</details>

---

## Step 1: Namecheap + FritzBox (one-time, before deploying Traefik)

### Namecheap

| Step | Where | Action |
|------|-------|--------|
| Enable DDNS | Domain List → Manage → Advanced DNS → Dynamic DNS | Toggle on, note the DDNS password |
| Enable API | Profile → Tools → API Access | Enable, whitelist home IP (`curl -4 -s https://icanhazip.com`) |
| Add A record | Domain List → Manage → Advanced DNS | Type: A, Host: `vpn`, Value: current home IP |

### FritzBox

Navigate to **http://fritz.box → Internet → Permit Access → Port Sharing**, target `192.168.178.20`:

| Protocol | Port | Purpose |
|----------|------|---------|
| TCP | 443 | Traefik HTTPS / Headscale coordination |
| TCP | 80 | Let's Encrypt HTTP fallback |
| UDP | 3478 | STUN for Tailscale NAT traversal |

FritzBox automatically creates both an IPv4 NAT rule and an IPv6 firewall allow rule from each port sharing entry.

---

## Step 2: Mini PC — one-time host prep

```bash
# Install SSH server if not present
apt install openssh-server -y

# Create deploy user
useradd -m -s /bin/bash philipp
usermod -aG sudo philipp

# Add your SSH public key
mkdir -p /home/philipp/.ssh
echo "YOUR_PUBLIC_KEY" >> /home/philipp/.ssh/authorized_keys
chmod 700 /home/philipp/.ssh && chmod 600 /home/philipp/.ssh/authorized_keys
chown -R philipp:philipp /home/philipp/.ssh
```

---

## Step 3: Configure Ansible inventory and variables

1. Edit `ansible/inventory/hosts.yml` to set the Mini PC's IP and connection details.
2. Edit `ansible/group_vars/all/vars.yml` to change service versions or other non-secret config.

---

## Step 4: Deploy HashiCorp Vault first (chicken-and-egg)

Vault is deployed before it can hold the other service secrets. On first run, Ansible uses a root token you provide directly:

```bash
# Deploy just the Vault stack (no secrets needed yet — only the OIDC secret is fetched,
# and that comes later in the post-deploy wiring step)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml --tags hcvault
```

**Initialize Vault (one-time, immediately after first deploy):**

```bash
ssh philipp@192.168.178.20
docker exec -it vault vault operator init

# Save the 5 unseal keys and root token in Vaultwarden immediately
# Vault needs 3 of 5 keys to unseal after every restart
docker exec -it vault vault operator unseal  # run 3 times with different keys
```

Consider configuring auto-unseal later (cloud KMS or a dedicated key) to avoid manual unsealing after reboots.

---

## Step 5: Populate secrets in HashiCorp Vault

All secrets live at `secret/ansible` in Vault. Run this from your Mac after `vault login -method=oidc`.

**Secrets you can set immediately** (before any service is deployed):

```bash
vault kv put secret/ansible \
  minipc_passwd="..." \
  namecheap_api_user="..." \
  namecheap_api_key="..." \
  namecheap_ddns_password="..." \
  traefik_dashboard_auth="..." \
  pihole_webpassword="..." \
  keycloak_admin_user="admin" \
  keycloak_admin_password="..." \
  keycloak_db_password="..." \
  hcvault_oidc_secret="PLACEHOLDER" \
  vaultwarden_db_password="..." \
  vaultwarden_admin_token="..." \
  vaultwarden_oidc_secret="PLACEHOLDER" \
  headscale_db_password="..." \
  headscale_oidc_secret="PLACEHOLDER" \
  headplane_api_key="PLACEHOLDER" \
  headplane_cookie_secret="..." \
  gitlab_root_password="..." \
  gitlab_oidc_secret="PLACEHOLDER" \
  gitlab_api_token="PLACEHOLDER" \
  harbor_admin_password="..." \
  harbor_db_password="..." \
  harbor_oidc_secret="PLACEHOLDER" \
  homepage_oidc_secret="PLACEHOLDER" \
  homepage_nextauth_secret="..." \
  grafana_admin_password="$(openssl rand -base64 32)" \
  grafana_oidc_secret="PLACEHOLDER" \
  paperless_db_password="$(openssl rand -base64 32)" \
  paperless_secret_key="$(openssl rand -base64 50)" \
  paperless_admin_password="$(openssl rand -base64 32)" \
  paperless_oidc_secret="PLACEHOLDER" \
  paperless_api_token="PLACEHOLDER" \
  jellyfin_oidc_secret="PLACEHOLDER" \
  jellyfin_api_key="PLACEHOLDER"
```

Fields marked `PLACEHOLDER` are filled in during post-deploy wiring steps below.

<details>
<summary>Password generation reference</summary>

```bash
# Strong random password (use for any "..." above)
openssl rand -base64 32

# Vaultwarden admin token
openssl rand -hex 32

# Traefik dashboard auth (needs apache2-utils)
htpasswd -nB philipp

# Namecheap API key: Profile → Tools → API Access
# Namecheap DDNS password: Domain List → Manage → Advanced DNS → Dynamic DNS
# OIDC secrets: filled in after Keycloak is running (copy from Keycloak client credentials)
# headplane_api_key: generated after Headscale is running (see post-deploy section)
# gitlab_api_token: generated after GitLab is running (User Settings → Access Tokens)
```

</details>

---

## Step 6: Wire SSO (post-Keycloak)

After Keycloak is running, create OIDC clients for each service before deploying them:

1. Log into `https://auth.philippthesurfer.com` → create realm `homelab`
2. Create OIDC clients: `headscale`, `hcvault`, `gitlab`, `harbor`, `vaultwarden`, `homepage`
3. Copy each client secret from Keycloak → Credentials tab, then add to Vault:

```bash
vault kv patch secret/ansible \
  hcvault_oidc_secret="..." \
  headscale_oidc_secret="..." \
  gitlab_oidc_secret="..." \
  harbor_oidc_secret="..." \
  vaultwarden_oidc_secret="..." \
  homepage_oidc_secret="..." \
  grafana_oidc_secret="..." \
  paperless_oidc_secret="..." \
  jellyfin_oidc_secret="..."
```

4. Re-run the affected roles to apply SSO config.

---

## Step 7: Generate post-deploy secrets

**Headplane API key** (after Headscale is running):

```bash
# Generate a Headscale API key for Headplane to manage nodes
ssh philipp@192.168.178.20
docker exec -it headscale headscale apikeys create --expiration 999d
```

Store it in Vault and re-run the headscale role:

```bash
vault kv patch secret/ansible headplane_api_key="<key-from-above>"
ansible-playbook ansible/site.yml --tags headscale
```

**Tailscale subnet router** (after Headscale is running):

Install the Tailscale client on the Mini PC:

```bash
ansible-playbook ansible/site.yml --tags tailscale-client
```

Then SSH into the Mini PC and authenticate manually:

```bash
tailscale up --login-server=https://vpn.philippthesurfer.com \
             --advertise-routes=192.168.178.0/24 \
             --hostname=minipc
# Opens a URL — visit it on any device and log in with your Keycloak account
```

Approve the advertised subnet route:

```bash
docker exec -it headscale headscale routes list
docker exec -it headscale headscale routes enable -r <route-id>
```

On any remote device that should reach internal services over VPN:

```bash
tailscale up --accept-routes
```

---

**GitLab API token** (after GitLab is running):

Log in to GitLab → User Settings → Access Tokens → create a token with `api` scope. Store it in Vault:

```bash
vault kv patch secret/ansible gitlab_api_token="<token>"
```

---

## Step 8: Migrate this repo to GitLab

```bash
git remote add homelab https://gitlab.home.philippthesurfer.com/<user>/homelab.git
git push homelab main
```

From this point, GitLab CI handles re-deploys on push. Ansible on the local machine is only needed for host-level changes.
