## Bootstrap

### Prerequisites (control machine)

```bash
# macOS: prevent "A worker was found in a dead state" fork-safety crash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES  # add to ~/.zshrc to make permanent

# Install hvac into Ansible's Python — required for community.hashi_vault lookups
# Pin to 1.x: hvac 2.x changed response types, breaking community.hashi_vault 7.x lookups
/opt/homebrew/Cellar/ansible/*/libexec/bin/python -m pip install "hvac"

# Install Ansible collections
ansible-galaxy collection install -r ansible/requirements.yml
```

### Deployment order

Infrastructure must be bootstrapped in dependency order. Priority reflects architectural importance; deployment order reflects technical dependencies.

```
[infrastructure]  common     → Docker, UFW, deploy user (prerequisite for everything)
[infrastructure]  traefik    → SSL and routing (prerequisite for all HTTPS services)

[priority 1]      keycloak   → SSO — must exist before any service is wired to it

[priority 2]      headscale  → VPN — wire Keycloak OIDC, then port forward goes live

[priority 3]      vault      → HashiCorp Vault — secrets automation
[priority 3]      vaultwarden → Password manager

[priority 4]      pihole     → Internal DNS (or migrate existing Pi-hole to IaC)
[priority 4]      gitlab     → Push this repo here; CI takes over re-deploys
[priority 4]      harbor     → Registry
```

<details>
<summary>exec commands for deployments</summary>

```bash
# Pre-requisite: authenticate to HashiCorp Vault (sets ~/.vault-token, valid 8h)
export VAULT_ADDR=https://vault.home.philippthesurfer.com
vault login -method=oidc

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

# 4. Headscale (wire Keycloak OIDC first — see post-bootstrap below)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags headscale

# 5. HashiCorp Vault
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags hcvault

# 6. Vaultwarden
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags vaultwarden

# 7. Pi-hole (migrate existing or fresh deploy)
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags pihole

# 8. GitLab + Harbor
ansible-playbook ansible/site.yml -i ansible/inventory/hosts.yml \
  --tags gitlab,harbor
```

</details>

### Pre-bootstrap: Namecheap + FritzBox (one-time, before deploying Traefik)

#### Namecheap

| Step | Where | Action |
|------|-------|--------|
| Enable DDNS | Domain List → Manage → Advanced DNS → Dynamic DNS | Toggle on, note the DDNS password |
| Enable API | Profile → Tools → API Access | Enable, whitelist home IP (`curl -4 -s https://icanhazip.com`) |
| Add A record | Domain List → Manage → Advanced DNS | Type: A, Host: `vpn`, Value: current home IP |

#### FritzBox

Navigate to **http://fritz.box → Internet → Permit Access → Port Sharing**, target `192.168.178.20`:

| Protocol | Port | Purpose |
|----------|------|---------|
| TCP | 443 | Traefik HTTPS / Headscale coordination |
| TCP | 80 | Let's Encrypt HTTP fallback |
| UDP | 3478 | STUN for Tailscale NAT traversal |

FritzBox automatically creates both an IPv4 NAT rule and an IPv6 firewall allow rule from each port sharing entry.

### Post-bootstrap: wire SSO

After Keycloak is running, create OIDC clients for each service before deploying them:

1. Log into `auth.philippthesurfer.com` → create realm `homelab`
2. Create OIDC clients: `headscale`, `gitlab`, `harbor`, `vaultwarden`
3. Add each client secret to HashiCorp Vault:
   ```bash
   vault kv patch secret/ansible \
     headscale_oidc_secret="..." \
     gitlab_oidc_secret="..." \
     harbor_oidc_secret="..." \
     vaultwarden_oidc_secret="..."
   ```
4. Re-run the affected roles to apply SSO config

### Pre-deploy: Harbor internal secrets (one-time, before first Harbor deploy)

Harbor requires several internal secrets in addition to its OIDC secret. Generate them all at once:

```bash
vault kv patch secret/ansible \
  harbor_admin_password="$(openssl rand -base64 24)" \
  harbor_db_password="$(openssl rand -base64 24)" \
  harbor_core_secret="$(openssl rand -base64 32)" \
  harbor_jobservice_secret="$(openssl rand -base64 32)" \
  harbor_csrf_key="$(openssl rand -hex 16)" \
  harbor_registry_http_secret="$(openssl rand -base64 32)"
```

| Secret | Purpose |
|---|---|
| `harbor_admin_password` | Initial local admin password |
| `harbor_db_password` | PostgreSQL password for harbor-db |
| `harbor_core_secret` | Shared HMAC secret between harbor-core, registryctl, and jobservice |
| `harbor_jobservice_secret` | Authentication secret for jobservice → core callbacks |
| `harbor_csrf_key` | 32-char key for CSRF token signing in harbor-core (Beego) |
| `harbor_registry_http_secret` | HTTP secret shared between the OCI registry and registryctl |
| `harbor_oidc_secret` | Keycloak client secret (set during SSO wiring above) |

### Post-bootstrap: initialize HashiCorp Vault

HashiCorp Vault requires a one-time init after first deploy:

```bash
ssh deploy@192.168.178.20
docker exec -it vault vault operator init

# Save the 5 unseal keys and root token in Vaultwarden immediately
# Vault needs 3 of 5 keys to unseal after every restart
docker exec -it vault vault operator unseal  # run 3 times with different keys
```

Consider configuring auto-unseal later (e.g. using a cloud KMS or a dedicated key) to avoid manual unsealing after reboots.

### Post-bootstrap: migrate this repo to GitLab

```bash
git remote add homelab https://gitlab.home.philippthesurfer.com/<user>/homelab.git
git push homelab main
```

From this point, GitLab CI handles re-deploys on push.

---
